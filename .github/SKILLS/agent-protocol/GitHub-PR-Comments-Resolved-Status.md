# GitHub PR 评论 resolved 状态查询

## 问题描述

通过 `gh api repos/<owner>/<repo>/pulls/<num>/comments` 获取的评论数据**不包含 resolved 状态**。

## 根本原因

- `pulls/comments` API 返回的是 **Review Comments**，每个评论独立存在
- GitHub 的 resolved 状态是**线程级（thread-level）**属性，不是评论属性
- 只有同一线程内的评论被标记 resolved 时，整个线程才会显示为 resolved

## 解决方案

### 方法：GraphQL 查询 reviewThreads

使用 GitHub GraphQL API 的 `reviewThreads` 字段获取线程级 resolved 状态：

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

### 解析示例

```bash
# 获取未解决的评论
gh api graphql -f query='...' | jq '.data.repository.pullRequest.reviewThreads.nodes[] | select(.isResolved == false) | .comments.nodes[0]'
```

## 关键字段

| GraphQL 字段 | 说明 |
|--------------|------|
| `reviewThreads.nodes[].isResolved` | 线程是否已解决 |
| `reviewThreads.nodes[].comments.nodes[].id` | 评论 ID |
| `reviewThreads.nodes[].comments.nodes[].body` | 评论内容 |
| `reviewThreads.nodes[].comments.nodes[].path` | 文件路径 |
| `reviewThreads.nodes[].comments.nodes[].createdAt` | 创建时间 |

## 经验总结

1. **REST API vs GraphQL**：获取 resolved 状态必须用 GraphQL
2. **缓存问题**：添加 `--cache 0` 避免缓存干扰
3. **评论定位**：通过 `createdAt` 时间戳与评论内容定位具体问题
4. **线程结构**：GitHub 评论是按线程组织的，同一线程的评论共享 resolved 状态
