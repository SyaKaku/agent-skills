# 低层派发启动核验

仅在动态指南证明 `worker-start` 无法表达所需拓扑、父级
[`SKILL.md`](../SKILL.md) 已选择 `dispatch --inject` 回退时读取。`injected: true` 只表示
运行时接受了终端注入，不证明 agent 已提交 prompt。

1. 派发前用 `terminal read` 记录 `nextCursor`；派发后读取该 cursor 的增量和当前
   terminal 尾部，并用短时、带显式 `--timeout-ms` 的
   `terminal wait --for tui-idle` 检查状态。
2. 所选 worker 手册列出的本轮提交信号，以及新的 agent 输出、工具调用或 lifecycle
   消息，均可证明已启动；此时保持只读。`terminal wait` 返回 `satisfied=false` /
   `blockedReason=codex-interactive-prompt` 通常不代表 idle。只有 terminal 尾部明确
   停在当前 dispatch 的 `[Pasted Content <N> chars]`、没有审批/问答菜单，且两次读取
   的 cursor 均未增长时，才记录“当前 prompt 待提交”。其他状态继续观察并保存原始
   结果。
3. 仅当当前 dispatch 仍为 `dispatched`，且 `tui-idle` 明确满足或已取得“当前 prompt
   待提交”信号，并在有界观察后没有本轮提交或活动证据时，补发一次。除 terminal
   尾部已有明确待提交输入外，至少在两次 `terminal read` 之间完成一个短时、带显式
   `--timeout-ms` 的 `check --wait` 窗口，并确认两次 `nextCursor` 均未增长。若返回
   Delivery，立即按父级 Delivery 流程处理，不得把未处理消息当作静默：

   ```text
   orca terminal send --terminal <worker-handle> --text '' --enter --json
   ```

4. 补发后重新读取增量并确认 worker 已启动。仍无活动证据时保存 `dispatch-show` 与
   terminal 输出并升级；同一 dispatch 不得再次补回车、重新 `task-create` 或重复
   `dispatch`。

只认派发前 cursor 之后的本轮证据；复用 terminal 时不得把旧工单的输出或 hook 当作
当前 dispatch 已启动。
