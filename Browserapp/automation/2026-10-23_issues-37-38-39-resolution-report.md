# GitHub Issues #37, #38, #39 深度排查与修复完成报告

- **代码仓库**: `/Volumes/ssd1t/claude/OpenBrowser/openbrowser`
- **解决的问题**:
  - [Issue #37: 能否增加一个回收站的功能？](https://github.com/sheying2013/OpenBrowser/issues/37)
  - [Issue #38: 发现一个BUG - 电脑关机了，第二天打开的时候经常会有一些环境打不开，是没有正常退出指纹浏览器的原因吗？](https://github.com/sheying2013/OpenBrowser/issues/38)
  - [Issue #39: 同样的sk5代理 比特浏览器速度很快，在open浏览器，网页都打不开](https://github.com/sheying2013/OpenBrowser/issues/39)
- **验证结论**: 自动化回归套件专项通过（`selftest:issues` 全绿），全量 UI 与隔离自检 90/90 项无回退通过，收尾自检 3 轮全面检查零问题放行。

---

## 一、 问题根因深度剖析

### 1. Issue #39: SOCKS5 代理速度慢、网页（如 Twitter）无法打开
- **根因分析**:
  1. **本地 DNS 污染与解析超时（核心根因）**:
     在原工程 `Browserapp/engine.js:3846` 中，系统仅在 `proxyConfig?.authenticated`（即带有用户名密码）时才调用 `startAuthenticatedProxy` 启动本地 HTTP CONNECT 桥接。对于**无认证的商业 SOCKS5 代理**（例如用户反馈的 `206.232.73.52:7622`），原系统直接生成 `--proxy-server=socks5://206.232.73.52:7622` 传给 Chromium。Chromium 在接受裸 `socks5://` 协议头时，默认会在**本机操作系统层面进行域名 DNS 解析**，然后再将解析出的 IP 传给 SOCKS5。在国内网络环境下，Twitter/X 等海外域名在本地解析直接遭到 DNS 污染阻断或超时，而比特浏览器等商业防关联工具强制走 SOCKS5 远端域名解析（即 Socks5h，ATYP 0x03），由代理服务器在境外做 DNS 解析。
  2. **QUIC / HTTP3 UDP 丢包卡顿**:
     部分商业 SOCKS5 代理节点不支持或故意限速 UDP 转发。Chromium 默认会并发尝试 QUIC (HTTP/3)，在代理不支持 UDP 时反复等待握手超时才降级回 TCP，导致页面首屏加载极慢。

### 2. Issue #38: 异常关机或直接断电后，第二天部分环境无法启动
- **根因分析**:
  1. **跨开机周期 PID 碰撞与伪锁阻断**:
     在 `Browserapp/automation/isolation.js:287` 中，启动时若发现存在锁文件 `openbrowser.lock`，会检查锁中记录的 `pid` 是否存活。电脑直接关机重启后，系统分配 PID 重新从 1 开始累加，原先崩溃留下的旧 PID 极容易撞上新开机系统的系统服务 PID（或因为权限不足抛出 `EPERM`），导致系统误判为“环境正在运行中 (`PROFILE_LOCKED`)”或“进程归属未确认拒绝解锁 (`PROFILE_LOCK_UNRECOVERABLE`)”，环境被永久锁死。
  2. **断电导致配置文件或锁文件截断损坏（0 字节/非法 JSON）**:
     断电瞬间磁盘写入被强制截断，导致 `Preferences` 或 `openbrowser.lock` 出现损坏，启动读取时抛出未捕获的 `SyntaxError` 异常导致流程终止。
  3. **Chromium 原生单例锁残留**:
     突然掉电导致用户数据目录下的 `SingletonLock`、`SingletonSocket`、`lockfile` 未被 Chromium 退出钩子清理，下次启动时 Chromium 自身检测到锁发生死锁。

### 3. Issue #37: 增加环境回收站功能（防止跨分组误删、支持还原与彻底删除）
- **需求与架构设计**:
  1. 原系统删除操作直接从内存列表删除环境并递归删除磁盘数据，一旦误操作跨分组多选删除，数据彻底丢失无法追回。
  2. 需要支持两阶段软删除机制：默认删除操作移入“回收站”保留数据目录；在回收站视图中支持“还原环境”、“彻底删除”与“清空回收站”。

---

## 二、 修复方案与工程落地

### 1. Issue #39 (SOCKS5 强制远程 DNS 与防卡顿)
1. **统一代理桥接判断 (`Browserapp/proxy-forwarder.js`)**:
   定义并导出 `needsProxyBridge(config)` 函数：
   ```javascript
   function needsProxyBridge(config) {
     if (!config || typeof config !== 'object') return false;
     if (Boolean(config.authenticated)) return true;
     // 无论是否有认证，所有 SOCKS5 代理必须走本地桥接以强制 Socks5h 远端 DNS 解析
     if (String(config.protocol || '').toLowerCase() === 'socks5') return true;
     if (process.env.OPENBROWSER_ALWAYS_BRIDGE_PROXY === '1') return true;
     return false;
   }
   ```
2. **Chromium 启动参数强制远程 DNS 与禁用 QUIC (`Browserapp/engine.js`)**:
   - 启动前将所有 SOCKS5 代理路由至本地本地代理桥（Node 桥或 Rust 核心），Chromium 仅与本地 HTTP CONNECT 端口通信，本地桥向上游发起 RFC 1928 SOCKS5 连接时一律传递域名（ATYP 0x03），彻底绕过本地被污染的 DNS。
   - 增加并固化 `--disable-quic`、`--disable-domain-reliability` 等启动参数，杜绝因 UDP 丢包导致的加载停滞。

### 2. Issue #38 (跨关机自愈恢复与锁保护)
1. **开机时间戳对比校验 (`Browserapp/automation/isolation.js`)**:
   利用 `os.uptime()` 计算当前操作系统的开机时刻 `systemBootTimeMs = Date.now() - (os.uptime() * 1000)`。若锁文件的修改时间或内容记录的创建时间早于系统开机时间，则在数学和操作系统原理上证明创建该锁的进程绝对已经死亡（随系统关机终结），直接安全抹除残留死锁，杜绝因 PID 碰撞造成的锁死。
2. **损坏配置自我修复 (`Browserapp/automation/profile-file-consistency.js`)**:
   在 `readPreferences` 中拦截 `SyntaxError`，自动将断电截断损坏的文件重命名为 `Preferences.corrupted.<timestamp>` 供备查，并返回安全默认空对象，触发 Chromium 的自愈重建逻辑。
3. **残留单例清理 (`Browserapp/engine/browser-candidates.js` & `Browserapp/engine.js`)**:
   在环境启动前增加针对 `SingletonLock`、`SingletonSocket`、`lockfile` 的开机时间检测与死锁清理。

### 3. Issue #37 (环境回收站完整功能)
1. **数据模型扩展 (`Browserapp/engine/profile-sanitizer.js` & `Browserapp/engine.js`)**:
   - Profile 字段增加 `trashed: Boolean` 和 `trashedAt: string | null`。
   - `BrowserEngine` 增加四组核心服务：
     - `trashProfiles(ids)`: 软删除，停用运行中环境，标记 `trashed: true` 并记录时间，保留本地目录。
     - `restoreProfiles(ids)`: 还原环境，清除软删除标记，回到正常列表。
     - `purgeProfiles(ids, deleteData)`: 彻底物理删除并递归清理数据目录。
     - `emptyTrash(deleteData)`: 一键清空回收站内所有环境。
2. **IPC 桥接暴露 (`Browserapp/main.js` & `Browserapp/preload.js`)**:
   注册并暴露 `profiles:trash`、`profiles:restore`、`profiles:purge`、`profiles:empty-trash`。
3. **列表过滤与隔离 (`Browserapp/ui/ui-profile-list.js`)**:
   - `filterProfiles` 增加逻辑：常规视图（全部、分组、未分组）硬隔离软删除环境，回收站（`activeGroupFilter === 'trash'`）仅呈现已删除环境。
   - 搜索过滤器补全 `profile.name` 检索支持。
4. **前端视图与交互联动 (`Browserapp/renderer.js` & `Browserapp/index.html`)**:
   - 分组筛选栏末尾新增带动态计数的“回收站”胶囊。
   - 回收站视图中，顶部工具栏替换为“还原选中”、“彻底删除”与“清空回收站”；表格操作列切换为“还原”与“彻底删除”按钮。
   - 批量删除对话框提供“移入回收站（安全）”与“彻底物理删除”双重选项。
5. **多语言国际化覆盖 (`Browserapp/i18n.js`)**:
   补齐全部 8 种语言的回收站相关字典（`trash.group_label`、`trash.empty_title`、`trash.restore_success`、`trash.purge_confirm` 等）。

---

## 三、 自动化测试与质量收尾证据

1. **新建专项集成测试**: `Browserapp/automation/issues-37-38-39-selftest.js`
   - 涵盖 SOCKS5 代理强制桥接判断（Socks5h 远端 DNS 解析机制）。
   - 涵盖模拟关机跨周期开机锁自动清理与断电截断文件优雅自愈。
   - 涵盖软删除、批量还原、单项清除、清空回收站与搜索隔离。
   - 执行结果：`ALL TESTS PASSED for GitHub Issues #37, #38, #39!`（退出码 0）。
2. **全量回归保障**:
   - `selftest:issues`: 100% 通过。
   - `selftest:secondaryui`: 90/90 项全绿通过。
   - `selftest:profile-list`: 通过。
   - `selftest:i18n`: 全语种字典无缺失通过。
   - `selftest:isolation`: 32/32 项通过。
   - `selftest:prefconsistency`: 5/5 项通过。
   - `selftest:rustrelease` & `selftest:version`: 版本一致性与规则检查全绿通过。
3. **收尾自检**:
   - `wrapup_check`: 连续 3 轮深度代码审查、构建验证与一致性检查零问题放行 (`verdict: fixed-point`)。
