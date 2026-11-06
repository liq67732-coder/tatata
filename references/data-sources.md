# 数据源取数指南

两个数据源：飞书（聊天记录）和 Moka（HR 系统）。取数只有一个原则：**能少取就少取**。取回来的每一条都要在内存里过一遍规则，取太多既慢又容易把噪音带进报告。

---

## 一、飞书

### 身份

统一用 `--as user`。`--as bot` 只能看到机器人自己所在的会话，看不到用户的工作群，且机器人对多数用户没有可用范围（报 `230013 Bot has NO availability to this user`）。

首次使用前确认：

```bash
lark-cli auth status
```

需要 `identities.user.status` 为 `ready`（或 `needs_refresh`，会自动刷新）。发消息另需 `im:message.send_as_user` 权限，缺了会报 `missing_scope`，按返回的 `hint` 走一次设备授权即可。

### 常用命令

```bash
# 1. 我参与的所有群（用于首次配置时让用户点名要监控哪些）
lark-cli im +chat-list --as user --types group --exclude-muted

# 2. 按群名找 chat_id
lark-cli im +chat-search --as user --query "招聘"

# 3. 增量拉某个群的消息
lark-cli im +chat-messages-list --as user \
  --chat-id oc_xxx \
  --start 2026-11-05T09:00:00+08:00 \
  --end 2026-11-06T09:00:00+08:00 \
  --order asc --page-all

# 4. 只看 @我 的消息（默认巡检首选，噪音最小）
lark-cli im +messages-search --as user --is-at-me \
  --start 2026-11-05T09:00:00+08:00 --end 2026-11-06T09:00:00+08:00 --page-all

# 5. 关键词扫描（每个关键词单独跑一次）
lark-cli im +messages-search --as user --query "怎么还没" \
  --start 2026-11-05T09:00:00+08:00 --end 2026-11-06T09:00:00+08:00
```

### 字段与坑

- 时间参数必须是带时区偏移的 ISO 8601（`+08:00`），不带偏移会被当成 UTC，导致窗口错 8 小时
- `--page-size` 最大 50，配合 `--page-all` 自动翻页；`--page-limit` 默认 10 页，群消息多的时候要调大
- 每条消息带 `sender_name`，不需要额外查通讯录；系统消息（`msg_type: system`）没有 sender，属正常
- 消息默认会附带 `reactions`，巡检场景可以加 `--no-reactions` 省流量
- 已撤回的消息不会出现在列表里，靠消息 id 回溯时可能查不到，属正常
- `--markdown @file` / `--file` 这类路径参数只接受**当前工作目录下的相对路径**。把报告写到 cwd 再用 `@reports/daily/xxx.md` 引用，传绝对路径会报 `unsafe file path`

### 关键词扫描的取舍

`feishu.keyword_watch` 里的词要选「带情绪或带催办意味」的短语，不要选「进度」「情况」这种高频词——后者会命中几十条无关消息，把日报淹掉。经验值：每个关键词每天命中 0-5 条是健康的，长期 > 20 条说明这个词太宽，换成更具体的短语。

---

## 二、Moka

Moka 通过 MCP 工具调用。**调用前先 `ToolSearch` 加载对应工具的 schema，不要凭名字猜参数**。

### 模块与工具映射

| 配置里的模块 | 用途 | 工具 |
|---|---|---|
| `recruiting_pipeline` | 各阶段候选人数量、卡在哪个环节 | `mcp__moka__list_pipeline_candidates`、`mcp__moka__get_recruiting_stage_statistics` |
| `candidate_stuck` | 候选人在某阶段停留时长 | `mcp__moka__list_pipeline_candidates`（取进入阶段时间） |
| `interview_feedback` | 面试安排与反馈提交情况 | `mcp__moka__list_interview_overview`、`mcp__moka__get_interview_records` |
| `job_health` | 职位在招状态、发布时长、简历量 | `mcp__moka__search_jobs` |
| `onboarding` | 待入职、入职流程进度 | `mcp__moka__get_onboarding_progress` |
| `attendance` | 迟到、缺卡异常 | `mcp__moka__get_team_attendance` |
| `approvals` | 待我审批 / 我发起被退回 | `mcp__moka__get_my_approvals` |
| `my_tasks` | 分配给我但未推进的候选人 | `mcp__moka__list_my_assigned_candidates` |

### 硬性约定

- **只调 `config.yaml` 里 enabled 的模块**。没开的模块一次都不要调，既省时间也避免越权
- 薪资、编制、身份证等敏感字段即使工具返回也不写入报告、不落盘
- 工具返回空列表时，报告里写「无数据」，不要写成「无异常」——两者含义完全不同，前者可能是权限或配置问题
- 接口报错时，日报里用一行灰字说明「Moka 今天的面试数据没取到」，不要把整个日报搞崩

### 首次试通

配置完后先跑一次只读的轻量查询（例如 `search_jobs` 取前几条），确认返回结构和预期一致，再写进规则。字段名对不上是最常见的翻车点，不要等到第一次正式巡检才发现。
