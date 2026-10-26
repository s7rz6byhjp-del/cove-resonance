# 网易云「一起听」NIM ChatRoom 414 修复交接

> 分支：`fix/nim-chatroom-414-handoff`
>
> 目标：保留 Cove Resonance 现有 MCP、Bridge 与网易云 HTTP 控制能力，修复实时 NIM ChatRoom 登录，使 iPhone 官方网易云 App 与服务端账号之间能够同步播放、控制播放并双向发送房间消息。

## 1. 当前实测现象

2026-10-26，在 iPhone 官方网易云 App 已发起「一起听」并邀请服务端账号后：

- Cove Resonance 能通过 HTTP 读取当前一起听房间；
- 能看到当前歌曲（实测为宇多田ヒカル《One Last Kiss》）及暂停状态；
- `netease_together_realtime` 连接失败；
- 播放控制未能完成；
- 服务端返回：`NIM chatroom auth failed code=414`。

这说明房间发现和网易云业务登录基本可用，故障集中在 NIM 实时链路的聊天室入房认证阶段，而不是 iPhone 邀请本身。

## 2. iOS 使用方式

不需要在 iPhone 安装 GitHub 项目、越狱或运行后台脚本。

- 用户端：iOS 官方网易云音乐 App，负责发起／加入一起听并播放音频；
- 服务端：VPS 上的 Cove Resonance，使用另一个网易云账号加入同一房间；
- Aru／MCP：调用服务端工具读取房间、控制播放、收发房间消息。

因此本次只修 VPS 的实时传输层，iOS 客户端保持不变。

## 3. 当前代码路径

重点文件：

- `src/netease/realtimeTransport.ts`
  - `NeteaseRealtimeTransport.connectInternal()`
  - `requestEnterTicket()`
  - `chatroom.enter()`
  - `sendChatRoomText()`
  - `buildRealtimeChatTextMessage()`
- `src/netease/nimTicketBootstrap.ts`
  - 使用 `node-nim` 创建 `NIMClient`／`NIMPlugin`
  - `NIMClient.login()` 后调用 `chatRoomRequestEnterAsync()` 获取入房 ticket
- `package.json`
  - 当前 `node-nim` 版本：`10.10.13`

当前流程：

1. 网易云业务接口取得 `accId`、IM token、`chatRoomId`；
2. 独立 child process 中用 `NIMClient.login()`；
3. `chatRoomRequestEnterAsync()` 取得 `requestLoginData`；
4. parent process 的 `ChatRoom.enter()` 入房；
5. `enter` 回调 step=5 返回 414。

注意：414 出现在第 4／5 步，不应仅凭错误码断言一定是 token 过期。需要记录并比较 SDK、appKey、accId、token 来源、ticket 内容结构、ChatRoom enter 参数与官方兼容实现；日志必须脱敏。

## 4. 已核实的参考实现

### A. folium-mod-music-party

- 仓库：https://github.com/tabidachinokaze/folium-mod-music-party
- 定位：Folia 桌面播放器插件；不是 iOS App，也不是可直接替换的 MCP。
- 可借鉴：
  - 与网易云官方一起听房间互通；
  - 同步播放；
  - 文字聊天、Emoji、图片、@、聊天历史；
  - 真实账号验证说明与消息协议实现。

### B. TinyCloudMusicDesktop

- 仓库：https://github.com/AcceleratorPan/TinyCloudMusicDesktop
- 参考文档：`docs/api-gap-tasks/P2-06_LISTEN_TOGETHER_REALTIME.md`
- 可借鉴：
  - 两个真实账号验证创建／加入一起听房间；
  - 播放、暂停、切歌、队列同步、断线恢复；
  - 使用与网易云兼容的 NIM Native SDK 路线。
- 限制：其目标不包含完整聊天室文字消息，不能单独满足本项目要求。

### C. olbyine-music

- 仓库：https://github.com/Huliang2597/olbyine-music
- 重点文件：
  - `app/src/main/kotlin/com/olbyine/music/data/ListenTogetherIm.kt`
  - `app/src/main/kotlin/com/olbyine/music/data/NeteaseClient.kt`
- 可借鉴：
  - 一起听播放控制；
  - ChatRoom 消息接收、发送和历史；
  - 消息扩展字段中的 `listenTogether`、`FRIEND`、`roomId` 等格式。

## 5. 推荐改造策略

不要重写整个项目。保留：

- HTTP 房间发现；
- MCP 工具定义；
- Bridge／Aru 消息转发；
- 播放状态模型、回执等待和队列校验；
- 现有消息 encode／decode 上层接口。

只替换或抽象实时底层：

```text
NeteaseRealtimeTransport
  ├─ NodeNimRealtimeAdapter（现有，保留作实验／回退）
  └─ NativeNimRealtimeAdapter（建议新增）
```

建议先让 transport 依赖稳定接口：

- `connect(options)`
- `disconnect()`
- `sendChatRoomText(text)`
- `onPlaybackEvent`
- `onChatMessage`
- `getStatus()`

然后把 NIM SDK 细节移到 adapter。优先在 Linux VPS 验证 SDK 的架构、glibc 和动态库兼容性；不要在主服务进程里反复 init／cleanup 原生 SDK。

## 6. 排查顺序

### 第一步：复现并收集脱敏诊断

记录：

- Node、OS、CPU 架构；
- `node-nim` 版本；
- NIM appKey 是否与参考实现一致；
- accId 是否为空、是否纯数字／字符串；
- token 长度、哈希前 8 位（不要打印 token）；
- chatRoomId；
- `chatRoomRequestEnterAsync()` 返回码；
- ticket 字节长度和 JSON 顶层键名（不要打印 ticket 正文）；
- `ChatRoom.enter()` 本地返回值；
- enter 回调各阶段的 step／code。

### 第二步：排除凭据生命周期问题

每次重新进入房间时：

- 重新从网易云接口取 `music_listenTogether` IM token／credential；
- 不复用上一次房间的 enter ticket；
- 确认 token 对应当前网易云账号、当前 appKey；
- 若 token 刷新后仍稳定 414，转向 SDK／参数兼容问题。

### 第三步：比较 SDK 与 enter 参数

逐项比对真实可用实现：

- Native SDK 版本；
- appKey；
- 登录 auth type／login extension；
- ChatRoom request-enter API；
- `ChatRoom.enter()` 的 login data 与 member info 结构；
- room id 使用 number 还是 string；
- SDK 初始化目录与权限；
- Linux 原生依赖。

### 第四步：替换实时 adapter

若 `node-nim` 在当前环境持续 414：

- 引入兼容的 NIM Native SDK／helper 进程；
- helper 只暴露本地 IPC，不暴露公网；
- 主服务通过 JSON lines／Unix socket 发送 connect、send、disconnect；
- helper 回传 playback、chat、status 事件；
- token、ticket 不进入普通日志、MCP 输出或模型上下文。

## 7. 消息与播放验收标准

必须使用两个真实网易云账号验收：iPhone 官方 App + VPS 服务账号。

### 入房

- iPhone 发起一起听并邀请服务账号；
- VPS 能识别邀请并进入同一 room；
- realtime 状态显示 connected；
- 断线后可以自动恢复，不产生重复成员或事件风暴。

### 播放控制

以下操作必须由 VPS／MCP 发出，并在 iPhone 官方 App 上真实生效：

- resume；
- pause；
- next；
- 切到指定的、已在队列中的歌曲；
- 插入下一首（若原工具支持）。

不能只以 HTTP 200 或本地状态改变作为成功；必须等实时回执，并再次读取房间状态校验。

### 双向聊天

- iPhone 发一条唯一文本，VPS 收到 sender、text、messageId；
- VPS 发一条唯一文本，iPhone 官方 App 可见；
- 发送消息带正确 `listenTogether` client extension；
- 自己发出的消息不会被 Bridge 重复触发；
- 重连后历史／实时消息不发生明显重复。

### 安全与回归

```bash
npm install
npm test
npm run typecheck
npm run build
```

并检查：

- 仓库和提交中无 Cookie、token、ticket、密码；
- 日志默认脱敏；
- HTTP-only 功能在实时链路失败时仍能正常读取状态并明确报告降级；
- iPhone 无需安装任何额外程序。

## 8. 建议交付物

VPS 修复完成后提交到本分支，至少包含：

1. realtime adapter／helper 源码；
2. Linux VPS 安装与原生依赖说明；
3. `.env.example` 新增项（仅变量名，不含真实值）；
4. 单元测试与双账号实测记录；
5. 一份脱敏故障日志；
6. 若使用闭源／不可再分发的 SDK，说明取得方式和许可证限制，不要把无权分发的二进制直接提交仓库。

## 9. 当前边界

本分支目前是修复交接分支，尚未声称已经解决 414。文档中的根因是待验证假设；只有完成上述双账号端到端验收，才能关闭问题。