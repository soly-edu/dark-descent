---
name: Ghost批注
description: 在正文中插入内联批注,标注设定冲突、角色不一致、伏笔账本问题等
pr_prefix: Ghost批注
---

## 执行规则

### 批注格式

批注以 HTML 注释格式插入正文:

```html
<!-- GHOST: [批注内容] -->
```

### 批注结构

```yaml
type: "setting_conflict" | "character_inconsistency" | "timeline_error" | "other"
severity: "info" | "warning" | "error"
message: "[人类可读的提示信息]"
entity_id: "[相关实体ID,可选]"
```

### 审查维度

#### 1. 设定冲突扫描

对比正文中的角色行为与 `[@角色:名]` 的 `绝对动机`,不一致时输出批注。

示例:
```html
"你早就知道？"她声音很轻,像在自言自语。
<!-- GHOST: warning | 设定冲突: 林奕(绝对动机: 自保优先)此处主动追问可能违背人设 -->
```

#### 2. 机制越界检测

检查正文是否违反 `[#机制:名]` 的 `绝对限制`。

示例:
```html
他凝聚魔力,火焰在指尖跳动。
<!-- GHOST: error | 机制越界: 当前世界无魔法设定,违反绝对限制 -->
```

#### 3. 伏笔账本校验

对照 `伏笔账本.yaml` 中 `status=pending` 的条目,确认本章是否:
- 无意中提前回收伏笔
- 与未回收的伏笔内容冲突

示例:
```html
<!-- GHOST: info | 伏笔提醒: 本段描述暴露了伏笔foreshadow_003的内容,但账本中建议回收章节为18章 -->
```

## 输出

- 注入批注后的 `正文手稿/第XXX章/正文.md`
- 批注汇总报告（附在PR描述中,按severity分类列出所有批注）
