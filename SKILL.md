---
name: weekly-review-tips
description: 每天扫描飞书聊天记录与 Moka HR 数据，按用户自定义的异常规则识别异常并主动推送提醒；每周汇总当周所有异常做归因分析、给出可执行建议并生成周报；通过建议台账与用户反馈持续调整判断阈值和建议策略。当用户提到「周周复盘」「异常提醒」「每日巡检」「本周异常」「周报」「盯一下招聘进度 / 面试反馈 / 考勤异常」时使用。
agent_created: true
---

# 周周复盘小贴士

## 概述

把「人肉记着要盯的事」变成一个每天主动汇报、每周主动复盘的小助手。核心链路是四段：

1. **采集** — 从飞书聊天记录和 Moka HR 系统拉取当日新增信息
2. **判定** — 按用户配置的异常规则匹配，命中即建档
3. **提醒** — 每天固定时间推送一份「今天需要你出手的事」清单
4. **复盘** — 每周把当周异常聚合成归因分析 + 建议，并把建议记入台账，靠用户反馈逐月校准

它只做「提醒 + 分析 + 建议」，**不代替用户给别人发消息、不催人、不改数据**。这条边界是本 skill 的第一原则。

## 何时使用

- 用户要求制造 / 维护「每天提醒异常 + 每周复盘建议」这类助手
- 用户说「帮我盯一下这周的招聘进度 / 面试反馈 / 考勤异常」
- 用户要求生成周报、周度复盘、异常汇总分析
- 定时任务（automation）触发，执行每日巡检或周度复盘

## 运行前检查

工作目录固定为 `~/.workbuddy/weekly-review/`，与 skill 目录分离，避免升级 skill 时丢数据：

```
~/.workbuddy/weekly-review/
├── config.yaml            # 用户配置（数据源、群名单、阈值、时间、边界）
├── state/
│   ├── last_run.json      # 上次巡检时间戳，决定本次数据窗口
│   ├── open_items.json    # 未闭环异常（含提醒次数、首次发现时间）
│   └── strategy.md        # 阈值调整与策略演进记录
├── ledger/
│   └── suggestions.csv    # 建议台账：日期/异常类型/建议/反馈/是否见效
└── reports/
    ├── daily/YYYY-MM-DD.md
    └── weekly/YYYY-Www.md
```

启动时按顺序检查：

1. `config.yaml` 是否存在 → 不存在则复制 `assets/config.example.yaml` 并逐项向用户确认（尤其「异常规则」和「提醒时间」两节必须有值）
2. `state/last_run.json` 是否存在 → 不存在说明是首次运行，把数据窗口设为「昨天 00:00 至今」，并在首份日报里说明这是基线期
3. 检查飞书与 Moka 的可用性 → 飞书用 `lark-cli auth status` 确认 user 身份 ready；Moka 用任一轻量查询试通即可

配置缺项时，不要猜着跑。缺哪项就只问哪项，一次问清，不要连环追问。

## 每日巡检

按以下顺序执行，一步一落盘：

**Step 1 确定数据窗口**

读 `state/last_run.json` 的 `last_run_at`，窗口 = `last_run_at` 至今。若窗口超过 48 小时（说明中间漏跑了），把窗口截断到 48 小时并在日报里注明「中间有漏跑，只回溯两天」。

**Step 2 拉飞书数据**

按 `config.yaml` 的 `feishu.scope` 决定范围：

```bash
# 只看 @我 的消息（最省、噪音最小）
lark-cli im +messages-search --as user --is-at-me --start <ISO8601> --end <ISO8601> --page-all

# 指定群的增量消息
lark-cli im +chat-messages-list --as user --chat-id oc_xxx --start <ISO8601> --end <ISO8601> --order asc --page-all

# 关键词命中（"催一下""没人回""怎么还没"等）
lark-cli im +messages-search --as user --query "催一下" --start <ISO8601> --end <ISO8601>
```

`feishu.keyword_watch` 里的每个关键词都要单独搜一遍，结果按 `message_id` 去重。命令细节见 `references/data-sources.md`。

**Step 3 拉 Moka 数据**

只调 `config.yaml` 中 `moka.enabled_modules` 列出的模块，未列出的不要调。工具与字段映射见 `references/data-sources.md`。返回为空不等于异常——空结果必须在日报里区分为「无数据」而不是「无异常」。

**Step 4 规则判定**

按 `references/anomaly-rules.md` 的规则库逐条判定，命中的生成异常条目：

```json
{
  "id": "<type>:<entity_id>",
  "type": "candidate_stuck",
  "entity": "张三 / Java 后端 / 二面",
  "severity": "high",
  "first_seen": "2026-11-03",
  "last_seen": "2026-11-06",
  "evidence": [{"source": "moka", "ref": "candidate_id=123", "detail": "二面完成 5 天未反馈"}],
  "remind_count": 1,
  "status": "open"
}
```

判定必须带证据（消息 id 或记录 id），没有可回指的原始数据就不要建条目——这是防幻觉的硬门槛。

**Step 5 去重与升级**

与 `state/open_items.json` 比对：

- 已存在且证据未变 → 不重复推送，只更新 `last_seen`
- 已存在但严重度上升或超期天数跨档 → 标为 `escalated`，可以再推一次
- 上次推送距今不足 24 小时 → 一律不推，除非 `escalated`
- 数据里已看不到该异常（如候选人已推进、审批已通过）→ 标 `resolved`，从 open 列表移出，但保留在当周统计里

**Step 6 生成并发送日报**

按 `references/report-templates.md` 的日报模板，控制在上限条数内（默认 5 条，`config.yaml` 可调）。超出的按严重度排序，剩余折叠成一句「另有 N 条低优先级，回复"看全部"展开」。

发送：

```bash
# 私聊推给本人
lark-cli im +messages-send --as user --user-id ou_xxx --markdown @report.md

# 或推到指定群
lark-cli im +messages-send --as user --chat-id oc_xxx --markdown @report.md
```

发送后写回 `state/last_run.json` 和 `state/open_items.json`，日报存 `reports/daily/`。

**一条异常都没有时**：仍然发一条两行的短消息（「今天没有需要你出手的事」+ 今日扫描覆盖面）。沉默会让人怀疑它是不是挂了。

## 周度复盘

触发时间按 `config.yaml` 的 `weekly.schedule`（默认周五 17:00 或周一 09:00）。

**Step 1 汇总**

取当周 `reports/daily/` 与 `open_items.json`，整理出：本周新增异常、本周闭环异常、跨周未闭环异常、按类型分布。

**Step 2 归因**

不写空话。每条归因要指向具体环节，例如：

- 「面试反馈延迟集中在周三面试的岗位，3 条里有 2 条卡在同一个面试官」——指向人
- 「候选人卡在 offer 审批，平均停留 6 天」——指向流程
- 「考勤异常集中在月末」——指向时间规律

归因必须有数据支撑，样本不足 3 条时直接写「样本太少，暂不归因」，不要编。

**Step 3 给建议**

- 最多 3 条，按「本周最该做的一件事」排序
- 每条建议四要素齐全：**动作 + 对象 + 时间点 + 判断成功的标准**
- 反例：「加强面试反馈管理」
- 正例：「今天下午把李四名下 2 个超 5 天未反馈的候选人逐个 @ 一次，本周内把他的平均反馈时长压到 2 天内」

**Step 4 回看上周**

读 `ledger/suggestions.csv` 中上周的建议，逐条判断是否执行、是否见效，写进周报「上周建议回看」段落。没执行的不要指责，分析为什么没执行——这比建议本身更有信息量。

**Step 5 发送与记账**

按 `references/report-templates.md` 的周报模板发送，同时把本周建议追加进 `ledger/suggestions.csv`（字段见 `references/growth-loop.md`），周报存 `reports/weekly/`。

## 成长循环

让建议越来越准的三个机制，缺一不可：

**1. 每周记账**

每条建议都进台账，带 `type`（对应异常类型）和 `outcome`（pending / worked / failed / ignored）。用户回复「有用」「没用」「没做」或给表情时更新对应行。用户不回复就在下下周的周报里轻量追问一次，只问一次。

**2. 阈值自校准（每 4 周）**

统计近 4 周每条规则的命中数与采用率：

- 某规则连续 2 周命中但用户全部忽略 → 判定为噪音，阈值 +1 天或直接降级为「观察项」（只在周报出现，不进日报）
- 用户手动补充的异常落在某规则之外 ≥ 2 次 → 该场景需要新增规则，主动提议并请用户确认
- 某类建议连续 3 次 `failed` → 换一种建议思路，不要重复同一句

调整必须写进 `state/strategy.md`，格式：`日期 | 规则 | 原阈值 | 新阈值 | 触发调整的依据`。让用户能看到它「凭什么改了」。

**3. 阶段性复盘（每 4 周）**

单独出一份「策略回顾」，只回答三个问题：这 4 周它报的准不准？建议有没有被执行？下一阶段该盯什么、不该盯什么？这份回顾要和用户确认后再改 `config.yaml`，不要自作主张。

详细操作见 `references/growth-loop.md`。

## 红线

违反以下任何一条，宁可少做事：

- **只提醒，不代发**。不替用户给别人发消息、不催人、不 @ 第三方，例外只有用户当次明确要求
- **不读敏感内容**。薪资、绩效、身份证、体检、家庭情况等字段即使接口能返回也不读；聊天中出现的个人隐私不摘抄进报告
- **报告脱敏**。对外可转发的周报里，候选人用「姓 + 岗位」表示，不写手机号、邮箱
- **可回指**。任何一条异常都要能追溯到原始消息或记录，做不到就不报
- **留存有期限**。日报保留 90 天、`open_items` 中已 resolved 的条目保留 30 天后清理，聊天原文不落盘（只存引用 id 和一句话摘要）

## 资源

- `references/data-sources.md` — 飞书与 Moka 的取数命令、字段含义、常见坑
- `references/anomaly-rules.md` — 异常规则库：默认规则、判定逻辑、阈值建议
- `references/report-templates.md` — 日报与周报的版式模板
- `references/growth-loop.md` — 建议台账字段、阈值校准算法、阶段复盘流程
- `assets/config.example.yaml` — 配置模板，首次运行时复制到 `~/.workbuddy/weekly-review/config.yaml`
