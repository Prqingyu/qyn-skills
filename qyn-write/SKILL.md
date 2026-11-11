---
name: qyn-write
description: 信息中心写入端口 qyn_apply 的形状速查 —— 登记作业、改课次、整周课表、调休停课、周期任务、规划规则、删条目的完整 JSON 示例，以及 report/check_this 怎么读。不确定写入口怎么写时加载。
whenToUse: 准备调 qyn_apply 但不确定各节的 JSON 形状（schedule / rules / list 怎么写、upsert 还是数组、身份键是什么），或写入被拒绝要看正确写法时
---

# 写入口 `qyn_apply` 形状速查

**唯一写入口**：把「要改什么」写成一份改写意图送进来 —— 课表改 `schedule`、
规则改 `rules`、作业/提醒/单次任务登记进 `list`（注册表）。
**改哪个文件由内容决定**，不由调用哪个命令决定。先全量校验再落盘：任一处不合规，一个字节都不写。

两种写法：**数组 = 整段替换**；**`{upsert, remove}` = 按身份增量**
（只改一两处就用增量，别为了改一节把整份课表搬回来）。

## 各节怎么写

```jsonc
// 登记作业（gist 一行可执行要点 + detail 放全部原始信息）
{"list":{"upsert":[{"task":"示例课程C 作业3","date":"2026-10-15","due_time":"上课前",
  "gist":"Pxx 第 1/2/3 题，一页A4，写学号姓名",
  "detail":{"pages":"Pxx","problems":["5","6","7","10"],"submission":{"paper":"一页A4"}}}]}}

// 改一节课的地点（身份 = date|periods）
{"schedule":{"sessions":{"upsert":[{"date":"2026-10-14","periods":"5-8",
  "course":"示例课程F","location":"示例教室X1"}]}}}

// 水课：标在**课程**上（用户 2026-10-16：「这种明显要跟着课程不是课次」）
//   ⇒ 标一次，该课**所有**课次（含以后课表里新加的）都算水课；不用每次课都标
{"schedule":{"courses":{"upsert":[
  {"course":"示例课程G","water":true}]}}}

// ★★ **导入新一周课表：用 sessions.upsert，不要用 dated_sessions！**
//    `dated_sessions` 是**整段替换**（把整张课表换掉）—— 用它导入新课表会把本周课次**整批删掉**
//    （用户 2026-10-16 报过：新课从 10/19 开始，结果"今天明明有课却被认为没课"）。
//    内核现在会**拦住**这种替换并告诉你改用 upsert；确实要整段替换才写 "replace_future": true。
{"schedule":{"sessions":{"upsert":[
  {"date":"2026-10-19","periods":"1-2","course":"示例课程H","location":"教学楼 示例教室X1"}]}}}

// 整周课表导入（periods 是行首作息表，照抄）
{"schedule":{"periods":{"1":"08:00-08:45"},
  "dated_sessions":[{"date":"2026-10-12","periods":"1-2","course":"示例课程H","location":"示例教室X1"}]}}

// 调休：停课 / 删除
{"schedule":{"sessions":{"upsert":[{"date":"2026-10-14","periods":"5-8",
  "course":"示例课程F","kind":"cancelled"}]}}}
{"schedule":{"sessions":{"remove":["2026-10-14|5-8"]}}}

// 提醒（「提醒我明天中午取快递」这类 **不算任务** 的）—— 独立存储，不进注册表/规划
//   身份 = 日期 + 时刻 + 原文；省略 time ⇒ 当天 09:00（内核默认，别自己编）
{"reminders":{"upsert":[{"date":"2026-11-09","time":"12:00","text":"取快递"}]}}
// 办完了：同 date/time/text 再写一次，只改 status
{"reminders":{"upsert":[{"date":"2026-11-09","time":"12:00","text":"取快递","status":"done"}]}}
// 删掉：{"reminders":{"remove":["2026-11-09|12:00|取快递"]}}
// 读：qyn_reminders（--due 只看"现在该说了"的）。★ 别写进 list.upsert —— 那会变成任务 ✗

// 预留时长：用户说"这件事/这类事要留多久"时写 estimate_minutes（分钟，5~600）
//   · 周期任务（"以后每次"）写在 periodic_tasks 那一条上 ⇒ 物化时带到每次发生
//   · 单次任务直接写在注册表条目上
//   ★ upsert 是**整条替换** ⇒ 只给新字段会把原有 gist/due_time 冲掉，必须把原字段一起带上
{"schedule":{"periodic_tasks":{"upsert":[{"task":"示例课程F",
  "follow_course":"示例课程F","due_time":"23:59",
  "gist":"当次实验的完整报告，课后开写","estimate_minutes":60}]}}}
{"list":{"upsert":[{"task":"单次的活","date":"2026-11-10","due_time":"23:59",
  "status":"pending","estimate_minutes":90}]}}

// 周期任务：锚到课次 / 日历规律（与上课无关）
{"schedule":{"periodic_tasks":{"upsert":[{"task":"示例课程A作业",
  "follow_course":"示例课程A","due_time":"23:59"}]}}}
{"schedule":{"periodic_tasks":{"upsert":[{"task":"示例课程D作业",
  "every":"weekly","weekday":3,"due_time":"23:59"}]}}}   // weekday ISO：1=周一…7=周日

// 删周期任务 / 加规则 / 删规则 / 删待办
{"schedule":{"periodic_tasks":{"remove":["示例课程A作业"]}}}
{"rules":{"planning_rules":{"upsert":[{"id":"rule-002","kind":"preference",
  "condition":"中午没课","body":"优先安排示例课程作业"}]}}}
{"rules":{"planning_rules":{"remove":["rule-002"]}}}
{"list":{"remove":["示例课程作业 HW#"]}}

// 规则的机械校验窗口（两种写法；kind 决定阻塞还是警告）
{"rules":{"planning_rules":{"upsert":[{"id":"rule-004","kind":"preference",
  "no_slots":["19:00-22:30","00:00-06:00"],"condition":"…","body":"…"}]}}}
{"rules":{"planning_rules":{"upsert":[{"id":"rule-007","kind":"preference",
  "before_first_class":{"max_start":"12:00"},"condition":"当天上午有课的早晨",
  "body":"上课前不安排任务，用户要睡觉"}]}}}
//   ↑ 窗口 = 00:00 到**当天第一节课开始**；上午有课（≤ max_start，默认 12:00）才生效；
//     没课 / 只有停课 / 算不出时刻 ⇒ 不生效也不报错。no_slots 的结束时刻允许 24:00。

// 同名多条（周期任务的不同次发生）⇒ 必须给消歧字段指认是哪一条
{"list":{"upsert":[{"task":"示例课程D作业","date":"2026-10-21",
  "anchored_to":"2026-10-21","status":"pending"}]}}
{"list":{"remove":[{"task":"示例课程D作业","anchored_to":"2026-10-15"}]}}
{"list":{"remove":["示例课程作业 HW#"]}}          // 只有一条同名时可省消歧字段
```

## 几条容易错的

· **`sessions` / `periodic_tasks` / `planning_rules` 的 upsert 是整条替换** ——
  改一个字段也要把原字段一起写回（否则会丢 `location` / `kind` 之类）。
  拿不准就先 `qyn_show` 或 `qyn_sessions` 读回原条目再改。
· **`list.upsert` 按 `task` 整条替换**，但**关系字段会自动继承**
  （`session_key` / `occurrence_of` / `anchored_to`）—— 不用自己补。
· **同名多条**（周期任务的不同次发生，例如每周三那份作业的两周）**必须给消歧字段**
  （`date` / `anchored_to` / `occurrence_of` / `session_key`）：`upsert` 和 `remove` 都一样。
  不给 ⇒ **被拒绝**并返回 `candidates`（以前是静默只改第一条 / 按名字全删，都是数据事故）。
  只有一条同名时不必给。要改某一次的**状态**用 `qyn_mark` 的 `date` 参数。
· **`priority` 别默认填**（只有用户**明确**说过"这事重要"才给）；
  需要提前准备的给 `lead_days:N`（你判断语义，内核做算术）。
· 图/话里读到的信息**全部**放进 `detail`（任意嵌套 JSON），不要压成一句话。
· 写错键名会**明确报错**（不认识的键），照着报错改即可。

## 返回值怎么看

`report` 必须看，尤其 `check_this`。`report` 按文件分（`list` / `schedule` / `rules`），
各给 `added` / `updated` / `removed` / `missing` / `replaced`；
**只改注册表就只看 `report.list`**（别的键是空的 ≠ 没写成）。

· `missing` = 你要删的没匹配上（名字或身份写错了）
· `replaced` = 同身份被覆盖且名字变了（可能误伤了别的东西）

这两样**不是错误**，写入照样完成 —— 但**几乎总是说明你写错了什么**，当场纠正并告诉用户。

★ 真的改了东西 ⇒ 返回值里带 `plan_invalidated: true`（**事实**：注册表变了）——
看到它就要**重排**再回答（见技能 `qyn-time` 的「规划：派生物」）。


★★ **任务有"开始"和"结束"时，两个都要写**（`start` 字段，2026-10-27 加的）

- `start`（+可选 `start_time`）= **最早可以开始**：材料周三才发 / 课上才布置 / 实验做完才能写
- `date` + `due_time` = **截止（DDL）**
- 「周三开始、周五结束」⇒ `{"task":"…","start":"<周三日期>","date":"<周五日期>","due_time":"23:59","status":"pending"}`
- **不写 `start`** = 现在就能开始（不设下限）—— 别为了"排得早"乱写它
- 排期只能落在 `[start, date]` 里：`qyn_review` 会把排在 `start` 之前的块报成 `before_start`；
  `start` 晚于 `date`（写反）会让 `qyn_anomalies` 报 `start_after_due`
- 只读视图（`qyn_context` 的 `active`）会带上 `starts_in` / `can_start` —— 直接照着说"3 天后才能开始"

