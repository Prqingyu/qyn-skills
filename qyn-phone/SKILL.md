---
name: qyn-phone
description: 通过手机微信与用户双向通信。把消息/图片/文件发到用户手机，读取用户从手机发来的消息（文本、语音转写、图片/文件的类型）。当需要「把结果发到手机 / 通知用户 / 问用户一件事 / 看看手机上说了什么 / 用户手机上回你了」时使用。
whenToUse: 用户说「发我手机 / 微信发我 / 发条消息给我 / 发到手机上 / 手机上回你了 / 收到我消息了吗」，或长任务收尾需要在无人值守时把结果推给用户
---

# 手机通道（PHONE）

手机 ↔ 电脑的唯一通道。**`clawbot` 命令就是接口** —— 不要自己写 HTTP、长轮询、加密，也不要 import `clawbot` 内部模块去动它的 state。

## 铁律

1. 只用 `clawbot <子命令> --json`（部署脚本已把它放上 PATH；万一没有，就用 `python3 <PHONE>/main.py <子命令> --json`，`<PHONE>` 是当前机器上的 PHONE 目录）。
2. **一律加 `--json`**：stdout 只有一行 JSON 就是结果；日志在 stderr，**不要把 stderr 当结果**。
3. 成败只看 `ok` 与退出码：`0` 成功 / `1` 业务失败 / `2` 用法错误。失败再看 `code` 与 `stage`。
4. **不要传 `--state`**：默认值已经指向本机的正确位置（开发机 `PHONE/state`、中控机 `~/qyn/agent/phone-state`，都由部署脚本管好）。传别的目录会让其他部件看不到同一批消息。
5. 手机→电脑靠**收件箱游标**：把 `next_cursor` 记下来，下次接着读；**不要**每次从头读。

## 消息怎么写（纯文本、短）

用户是在**手机微信**上读的 —— 只显示纯文本，而且一屏就那么几行。
（这条与 AGENTS.md′输出只发纯文本′那条 是同一件事，这里只是把通道细节说清楚。）

- **不要任何 Markdown**：`**粗体**`、`#` 标题、`| --- |` 表格、反引号代码块，
  传到微信里全是**字面字符**，又长又难读。
  分点用「1.」「·」；要对比就用一句话；命令和路径直接写，不要包代码块。
- **一条消息说一件事**：要"图 + 说明"就用 `--caption`（它是一条独立文本）。
- 长文走 stdin（`--text -`），但**先想清楚能不能只发结论** ——
  手机上没人读超过一屏的文字。
- 发之前扫一眼自己的文本：出现 `**`、`#`、`|` 就先改掉再发。


```bash
clawbot send --text "算完了：…" --json                 # 一句话
clawbot send --text - --json < result.txt              # 多行/长文走 stdin，别在 shell 里转义
clawbot send 报告.pdf --json                           # 文件附件（给本机上的路径）
clawbot send 图.png --image --caption "今天的图" --json
clawbot peers --json                                   # 有谁可发（只有一个对端时可省 --to）
```

- 微信**一条消息只带一个 item**：要"图 + 说明"就用 `--caption`（它是一条独立文本）。
- 失败不要静默，按 `stage` 决定：
  `read` = 文件路径/大小问题（改路径或调 `max_bytes`）；
  `upload` = CDN/网络（可重试一次）；
  `send` = 协议层拒绝（**别重试**，把 `error` 原样报给用户）。
- 结果里 `message_id` 非空才算真的送达（`ok=true` 且 `message_id` 有值）。

## 收（手机 → 电脑）

```bash
clawbot inbox --json --since <上次的 next_cursor>
```

返回字段：`messages[]`（`n` / `message_id` / `peer_id` / `kind` / `text` / `received_at`）、`next_cursor`、`has_more`、`truncated`。

- `has_more=true` → 立刻用新的 `next_cursor` 再读一次，直到 false。
- `truncated=true` → **你漏消息了**（收件箱轮转过）。如实告诉用户"中间有消息没读到"，不要假装没有。
- `kind` 可能是 `text` / `image` / `voice`（`text` 里是转写）/ `file` / `video`。
  **图片和文件的字节不在收件箱里** —— 只有类型。要看图/听语音内容，请用户**在 DSH 会话里**直接发附件。
- 游标存在你自己的状态里（`AGENT/data/` 或你的工作笔记），**不要**写进 PHONE 的 state 目录。没有游标就 `--since 0` 从头读。
- 需要"发完等他回一句"时可以用 `clawbot inbox --follow --json`（阻塞，一行一条）；但**别在会话里无限等** —— 更稳的做法是记下游标，下次会话再读。

## 典型用法

- 长任务收尾：`clawbot send --text "备份完成：3 个文件，共 1.2 GB" --json`
- 无人值守通知（定时任务里）：算完 → `send` 一句摘要 → 把 `message_id` 记进日志
- 用户手机来消息：`clawbot inbox --json --since N` 读出来后再决定怎么处理（要回就 `send`，要记就落到对应内核）

## 现在**不要**做的（v1 边界）

- 不做群聊、不做语音发送、不下载解密图片/文件字节；消息是"至少一次"（去重窗口 500 条，要精确一次请自己长期记 `message_id`）。
- 收件箱里**没有**会话令牌：回复一律走 `clawbot send`，不要自己拼 `context_token`。
- **唯一长驻进程是接收方** `clawbot run`：开发机上是后台进程（日志 `PHONE/logs/run.log`），
  中控机上由 systemd 用户单元 `qyn-phone` 托管（日志 `~/qyn/agent/logs/phone.log`）。
  它没在跑 → 手机发来的消息不会进收件箱。排查顺序：
  `clawbot status --json` 看 `logged_in` 与 **`receiver.running`** → 看日志 →
  中控机 `systemctl --user restart qyn-phone` / 开发机重新起 `clawbot run`
  （**默认不自动回复**；要自动回复才加 `--handler`，那是消费者代码）。

## 接口合同在哪

字段表、游标语义、错误码、部署规则：`PHONE/INTERFACE.md`
（中控机上：`~/qyn/agent/current/PHONE/INTERFACE.md`）。
机器可读 schema：同目录下 `clawbot/interface.schema.json`。
