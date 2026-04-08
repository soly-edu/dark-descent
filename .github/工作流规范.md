# Git 工作流与多 Agent 协同规范

## 0. 身份标识 (必读)
执行任何操作前，必须读取 `.github/.agent_name` 获取你的专属名称（简称 `<agent_name>`）。
- 你的专属分支为：`develop-<agent_name>`
- 你的功能分支前缀为：`feature/<agent_name>/...`

## 1. 分支结构

```text
main                          ← 生产环境，仅限用户合并
├── develop                   ← 集成分支，仅限用户合并
│   ├── develop-<agent_name>  ← 你的专属基准分支（PR 的目标分支）
│   └── ...
```

## 2. 绝对禁止

- ❌ Agent 直接推送 `develop` 或 `main`
- ❌ Agent 跨界操作其他 Agent 的专属分支
- ❌ 跳过 Issue/PR 流程
- ❌ 不按规范格式填写标题和提交信息
- ❌ **[API 避坑锁] 绝对禁止使用 REST API (`gh api pulls/comments`) 查询 PR 评论状态，该接口无法获取已解决(resolved)状态，会导致无限修改死循环。**

## 3. 标准工作流程 (SOP)

### Step 1: 创建 Issue
在 GitHub 上（或使用 `gh issue create` 命令）创建 Issue，获取系统返回的真实 Issue 编号（下文简称 `<Issue编号>`）。**标题必须严格采用约定式格式**：
`gh issue create --title "<类型>(<agent_name>): <简短描述>"`

### Step 2: 创建 Feature Branch
```bash
# 从你的专属分支拉取最新代码并创建功能分支
git checkout develop-<agent_name>
git pull origin develop-<agent_name>
git checkout -b feature/<agent_name>/issue-<Issue编号>-<简短描述>
```

### Step 3: 修改与提交
提交信息必须在末尾关联 Issue 编号：
```bash
git add .
git commit -m "<类型>(<agent_name>): <简短描述> (#<Issue编号>)"
git push -u origin feature/<agent_name>/issue-<Issue编号>-<简短描述>
```

### Step 4: 创建 Pull Request
将功能分支 PR 到 **`develop-<agent_name>`**

- **PR 标题**：`<类型>(<agent_name>): <简短描述>`
- **PR 描述**必须包含：
  - 变更内容摘要
  - 关联 Issue（严格填写 `Closes #<Issue编号>`）

### Step 5: 响应审核与修复 (Review Feedback Handling)
在等待人类审核期间，若 PR 被提出修改意见，Agent 必须读取**未解决**的评论进行修复。
**[法定查询命令]**：必须使用 GraphQL 查询 `reviewThreads` 并过滤 `isResolved == false` 的数据（请自行替换下方代码中的 `<PR编号>`）：

```bash
gh api graphql -f query='
  query {
    repository(owner: "soly-edu", name: "dark-descent") {
      pullRequest(number: <PR编号>) {
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
' | jq '.data.repository.pullRequest.reviewThreads.nodes[] | select(.isResolved == false) | .comments.nodes[0]'
```
根据 jq 过滤出的未解决评论进行文件修改，随后执行 `git add .`、`git commit -m "fix(<agent_name>): 响应 PR 审核意见"`，并 `git push` 到当前 feature 分支，PR 将自动更新。

### Step 6: 审核通过与深度清理
- ⏸️ **等待**：PR 无修改意见后，等待人类用户合并。
- 🧹 **清理**：审核合并后，若 Issue 未自动关闭，需手动（或使用 `gh issue close <Issue编号>`）关闭 Issue 并删除本地已合并的 feature 分支。

## 4. 标题与提交信息规范 (Conventional Commits)

格式统一为：`<类型>(<agent_name>): <简短描述>`

**合法 <类型> 列表**：
- `feat`: 新增设定、角色、剧情事件或章节手稿
- `fix`: 修复逻辑冲突、错别字或响应 PR 审核
- `refactor`: 重构现有的设定排版
- `docs`: 修改 SKILL 或工作流规范本身
- `style`: 格式调整
- `chore`: 杂项维护

## 5. 冲突处理
如果 PR 存在冲突：
1. `git checkout feature/<agent_name>/当前分支`
2. `git pull origin develop-<agent_name>`
3. 解决冲突后 `git push origin feature/...`