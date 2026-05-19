---
name: dingtalk-vip-digest
description: "钉钉特别关注人未读消息收集与智能摘要。从多个群中收集用户特别关注人（关注的领导或同事）发送的所有未读消息，按群分类整理，生成结构化的内容汇总和摘要报告（JSON + Markdown + HTML）。触发词：特别关注消息、关注人未读消息、VIP消息摘要、领导消息汇总、重要消息、dingtalk vip digest、关注消息报告、未读消息汇总。当用户需要快速了解特别关注人在各群中的动态、不想逐群翻看消息时使用。"
description_zh: "钉钉关注人消息摘要"
description_en: "DingTalk VIP Message Digest"
disable: false
agent_created: true
---

# 钉钉特别关注人未读消息收集与智能摘要

从钉钉多个群中收集用户特别关注人发送的所有消息，按群和发送者分类整理，生成结构化的内容汇总和摘要报告。

## When to use

- 用户需要快速了解特别关注人（领导/关键同事）在各群中的消息动态
- 用户不想逐群翻看未读消息，希望一键汇总
- 用户说"帮我看看关注的人发了什么"、"领导消息汇总"、"特别关注消息"、"VIP消息"等
- 用户需要定期（如每日早晨、下班前）查看特别关注人的消息摘要
- 用户说"生成消息报告"、"消息汇总报告"、"未读消息摘要"等

## 前置条件

1. 钉钉 CLI（dws）已安装且版本 >= 1.0.24
2. 用户已完成钉钉登录授权（`dws auth status --format json` 返回已登录）
3. 用户在钉钉客户端已设置了特别关注人

## Steps

### Phase 1：环境与登录检查

```
1. 检查 dws 版本：
   dws version --format json
   → 版本 >= 1.0.24，否则提示升级

2. 检查登录状态：
   dws auth status --format json
   → 已登录则继续
   → 未登录则按钉钉套件 Skill 的 Step 3 授权流程完成登录

3. 获取当前用户信息（记录 userId 和名称，后续排除自己的消息）：
   dws contact user get-self --format json
   → 提取 userId、nick
```

### Phase 2：收集特别关注人消息

```
1. 拉取特别关注人消息（核心命令）：
   dws chat message list-focused --limit 50 --format json

2. 检查分页：如果 hasMore=true，用返回的 nextCursor 继续翻页：
   dws chat message list-focused --limit 50 --cursor <nextCursor> --format json
   → 重复直到 hasMore=false 或累计消息 >= 200 条

3. 对收集到的消息进行结构化解析：
   - 提取字段：senderNick（发送者昵称）、senderUserId、conversationTitle（群名）、
     openConversationId、content/text（消息正文）、createTime（发送时间）、msgType（消息类型）
   - 按 openConversationId 分组（按群归类）
   - 每个群内按 senderUserId 分组（按人归类）
   - 按 createTime 排序（从旧到新）
```

### Phase 3：补充获取未读会话上下文（可选增强）

```
1. 获取未读会话列表：
   dws chat message list-unread-conversations --count 50 --format json

2. 将未读会话与 Phase 2 收集到的消息进行交叉匹配：
   - 标记哪些群的关注人消息来自未读会话
   - 记录每个未读会话的未读消息数

3. 如果用户指定了特定的群名或特定关注人：
   - 按群名：dws chat search --query "群名关键词" --format json → 获取 openConversationId
   - 按人名：dws contact user search --keyword "人名" --format json → 获取 userId
   - 拉取特定发送者消息：
     dws chat message list-by-sender --sender-user-id <userId> \
       --start "<今天00:00 ISO-8601>" --end "<当前时间 ISO-8601>" --limit 50 --cursor 0 --format json
```

### Phase 4：生成摘要报告

```
1. 对每个群的消息按以下维度分析：
   - 消息数量统计（每个关注人发了多少条）
   - 消息类型分布（文本/图片/文件/链接等）
   - 关键内容提取（去除系统消息和无意义内容）
   - 时间线梳理（按时间顺序还原对话上下文）

2. 用 AI 对每个群的消息生成摘要：
   - 1-2句话概括该群中关注人讨论的核心话题
   - 提取可能需要你关注或回复的事项（如 @了你、提出了问题、布置了任务）
   - 标注紧急程度（高/中/低）

3. 生成全局摘要：
   - 跨群汇总：今天哪些关注人最活跃
   - 待办事项提取：可能需要你跟进的行动项
   - 关键决策/通知提取：影响你工作的重要信息
```

### Phase 5：输出报告

报告输出为三种格式：

#### 5.1 JSON 数据文件（供网站前端消费）

将结构化数据写入 `report-data.json`：

```json
{
  "reportTime": "2026-05-17T23:45:00+08:00",
  "summary": "全局摘要文本",
  "totalMessages": 42,
  "totalGroups": 5,
  "totalVips": 3,
  "urgentItems": ["待办项1", "待办项2"],
  "groups": [
    {
      "groupName": "项目冲刺群",
      "openConversationId": "xxx",
      "isUnread": true,
      "unreadCount": 12,
      "messages": [
        {
          "sender": "张总",
          "senderUserId": "xxx",
          "content": "消息内容",
          "time": "2026-05-17T14:30:00+08:00",
          "msgType": "text"
        }
      ],
      "groupSummary": "该群主要讨论了...",
      "actionItems": ["需要你回复..."],
      "urgency": "high"
    }
  ],
  "vipActivity": [
    {
      "name": "张总",
      "userId": "xxx",
      "messageCount": 15,
      "activeGroups": ["项目冲刺群", "管理层群"]
    }
  ]
}
```

#### 5.2 Markdown 报告

生成格式化的 Markdown 摘要报告，包含：
- 报告头：生成时间、统计概览
- 紧急事项：需要立即关注的项目
- 按群分类的详细摘要
- VIP 活跃度排行
- 待办行动项清单

#### 5.3 HTML 报告（供本地网站展示）

将报告数据注入到本地 localhost:8080 网站可消费的格式。

### Phase 6：用户确认与交互

```
1. 向用户展示 Markdown 摘要报告
2. 询问是否需要：
   - 查看某个群的详细消息
   - 对某条消息进行回复
   - 将报告保存到文件
   - 启动本地网站查看可视化报告
```

## 输出文件规范

| 文件 | 路径 | 用途 |
|------|------|------|
| JSON 数据 | `{workspace}/dingtalk-vip-reports/report-{YYYY-MM-DD-HHmmss}.json` | 结构化数据，供前端渲染 |
| Markdown 报告 | `{workspace}/dingtalk-vip-reports/report-{YYYY-MM-DD-HHmmss}.md` | 文本阅读版 |
| 最新报告索引 | `{workspace}/dingtalk-vip-reports/latest.json` | 指向最新报告文件路径 |

## Pitfalls

- **消息内容可能为空或非文本**：msgType 为 image/file/link 等非文本类型时，content 字段可能为空或为 JSON 结构，需特殊处理提取描述信息
- **特别关注人为空**：如果用户未在钉钉客户端设置特别关注人，`list-focused` 会返回空列表，此时应提示用户先在钉钉 App 中设置特别关注
- **分页中断**：网络不稳定时翻页可能失败，需记录已收集消息的最后 cursor，支持断点续拉
- **消息去重**：分页边界可能出现重复消息，需按 messageId 去重
- **时区问题**：createTime 的格式可能是时间戳或字符串，统一转换为 ISO-8601 格式带时区信息
- **CJK 编码**：中文消息在 shell 中可能乱码，按钉钉套件的 CJK 安全策略处理
- **PAT 权限**：`list-focused` 可能需要额外 PAT 授权，按钉钉套件的 host-owned PAT 流程处理

## Verification

1. 执行 `dws chat message list-focused --limit 5 --format json` 确认能返回数据
2. 检查返回的消息中包含 senderNick、content、conversationTitle 等关键字段
3. 生成的 JSON 报告能被 `JSON.parse()` 正确解析
4. Markdown 报告包含完整的摘要结构（统计、分群摘要、行动项）
5. 报告文件成功写入到指定路径
