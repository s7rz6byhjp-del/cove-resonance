# NIM 414 VPS 修复清单

- [ ] 在 VPS 复现 `NIM chatroom auth failed code=414`
- [ ] 记录 Node／OS／架构／SDK 版本（脱敏）
- [ ] 每次入房重新获取 IM credentials 与 enter ticket
- [ ] 确认 appKey、accId、token 来源一致
- [ ] 比对 `chatRoomRequestEnterAsync` 和 `ChatRoom.enter` 参数
- [ ] 抽象 realtime adapter，避免改坏 MCP／Bridge 上层
- [ ] 必要时改用兼容 NIM Native SDK helper
- [ ] iPhone 发消息 → VPS 收到
- [ ] VPS 发消息 → iPhone 官方网易云可见
- [ ] VPS pause／resume／next → iPhone 真正执行
- [ ] 断线重连通过
- [ ] `npm test`
- [ ] `npm run typecheck`
- [ ] `npm run build`
- [ ] secret scan 与日志脱敏检查
- [ ] 补充双真实账号验收记录
