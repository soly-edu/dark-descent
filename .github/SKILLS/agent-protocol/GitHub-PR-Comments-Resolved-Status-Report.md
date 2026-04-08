# GitHub PR 评论 resolved 状态查询问题报告

## 问题背景

在处理 GitHub PR #13 的 review comments 时，需要区分哪些评论已解决、哪些未解决，以便针对性地修复。

## 问题现象

使用 REST API 查询评论时，所有评论的 `resolved` 状态都显示为 `false`，无法判断哪些已被用户标记为已解决。

```
gh api repos/soly-edu/dark-descent/pulls/13/comments --cache 0
```

输出示例：
```
3043244388 | resolved=false | "不是在里世界战斗时就已经完成..."
3043247225 | resolved=false | "跟化学无关"
...
```

**实际情况**：用户告知共有 36 条评论，其中 31 条已解决，仅 5 条未解决。

## 尝试过的方法

| 方法 | 结果 | 原因 |
|------|------|------|
| REST API `pulls/comments` | 所有 resolved=false | API 不返回 resolved 字段 |
| REST API `pulls/:number/reviews` | 无评论详情 | 返回的是 review 汇总，非 thread |
| 管道传 python 解析 | 管道关闭错误 | bash 管道机制问题 |
| 不带管道的 `gh api \| head` | 获取数据成功 | 数据本身没问题 |
| 直接读取本地文件 | 与远程一致 | 排除本地缓存问题 |

## 根本原因

**GitHub 的 resolved 状态是线程级（thread-level）属性，不是单个评论属性。**

```
REST API /pulls/comments
├── 返回每个独立评论
├── 每个评论有 id, body, path, created_at
└── ❌ 没有 resolved 字段

GraphQL reviewThreads
├── 返回评论线程
├── 线程有 isResolved 字段
└── ✅ 可以获取 resolved 状态
```

## 解决方案

### GraphQL 查询

```bash
gh api graphql -f query='
  query {
    repository(owner: "soly-edu", name: "dark-descent") {
      pullRequest(number: 13) {
        reviewThreads(first: 50) {
          nodes {
            isResolved
            comments(first: 5) {
              nodes {
                id
                body
                path
                createdAt
              }
            }
          }
        }
      }
    }
  }
'
```

### 解析未解决的评论

```bash
gh api graphql -f query='...' | jq '
  .data.repository.pullRequest.reviewThreads.nodes[] 
  | select(.isResolved == false) 
  | .comments.nodes[0]
'
```

## 实际修复的5个问题

| 时间 | 评论 ID | 问题描述 | 修复方案 |
|------|---------|----------|----------|
| 08:27 | PRRC_861bCkn | 存在一个奇怪文件名文件 | 删除误创建的文件 |
| 09:06 | PRRC_861bzC9 | 白露弹回不影响缝隙状态 | 明确缝隙维持依赖恶魔腐蚀，与白露无关 |
| 09:08 | PRRC_861b1nI | 叹息之壁被冲击瞬间的描述不够准确 | 改为"完全失去作用 → 修复但本源受损" |
| 09:10 | PRRC_861b4Id | 为什么白露以前没有遭遇怪物 | 补充两个条件：叹息之壁开门 + 恶魔聚集（需要林奕信号） |
| 09:12 | PRRC_861b6Yn | 形成条件与林奕自相矛盾 | 区分"叹息之壁开门"和"恶魔聚集"两个独立机制 |

## 经验总结

1. **API 选择**：获取 resolved 状态必须用 GraphQL 的 `reviewThreads`
2. **缓存问题**：使用 `--cache 0` 避免缓存干扰实时数据
3. **线程结构**：GitHub 评论按线程组织，同一线程共享 resolved 状态
4. **定位方法**：通过 `createdAt` 时间戳 + 评论内容定位具体问题

---

## 相关文档

- `GitHub-PR-Comments-Resolved-Status.md` - 快速参考指南
- `GitHub-PR-Comments-Resolved-Status-Report.md` - 本报告
