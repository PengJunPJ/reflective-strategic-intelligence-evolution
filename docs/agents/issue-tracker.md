# 议题跟踪：GitHub

本仓库的议题和规格说明存放在 GitHub Issues 中。所有相关操作均使用 `gh` 命令行工具。

## 操作约定

- **创建议题**：`gh issue create --title "..." --body "..."`。多行正文使用 heredoc。
- **读取议题**：`gh issue view <编号> --json number,title,body,labels,comments`。
- **列出议题**：使用 `gh issue list --state open --json number,title,body,labels,comments --jq '[.[] | {number, title, body, labels: [.labels[].name], comments: [.comments[].body]}]'`，并根据需要添加 `--label` 和 `--state` 筛选条件。
- **将议题设为父议题的子议题**：创建时使用 `gh issue create --parent <父议题> ...`，或在创建后使用 `gh issue edit <父议题> --add-sub-issue <子议题>`（需要 `gh` 2.94 或更高版本）。
- **阻塞关系**：优先使用 GitHub 原生议题依赖关系。创建时使用 `gh issue create --blocked-by <编号>,<编号> ...`，或使用 `gh issue edit <子议题> --add-blocked-by <阻塞议题>`。如果原生依赖不可用，则在子议题正文顶部写入 `Blocked by: #<编号>, #<编号>`。
- **统计未关闭的阻塞项**：`gh api repos/<所有者>/<仓库>/issues/<编号> --jq .issue_dependencies_summary.blocked_by`。结果为 `0` 表示可以开始处理。
- **评论议题**：`gh issue comment <编号> --body "..."`。
- **添加或删除标签**：`gh issue edit <编号> --add-label "..."` 或 `gh issue edit <编号> --remove-label "..."`。
- **关闭议题**：先使用 `gh issue comment <编号> --body "..."` 说明原因，再运行 `gh issue close <编号>`。

仓库位置从 `git remote -v` 推断。在仓库克隆目录中运行时，`gh` 会自动识别仓库。

只能向用户自己的仓库提交议题，不得向技能来源仓库提交。每次会话首次运行 `gh issue create` 前，必须使用 `gh repo view --json nameWithOwner` 检查目标仓库；如果目标不是用户自己的仓库，必须先向用户确认。

## 将拉取请求作为分流入口

**PRs as a request surface: no.**

如需把外部拉取请求也纳入议题分流，可将上面的配置标记改为 `yes`。

GitHub 的议题和拉取请求共用编号空间，因此单独出现的 `#42` 可能指向任意一种对象。先使用 `gh pr view 42` 检查拉取请求，如果不存在，再使用 `gh issue view 42` 检查议题。

## 当技能要求“发布到议题跟踪系统”时

创建一个 GitHub Issue。

## 当技能要求“获取相关工单”时

按照“读取议题”的约定读取对应 GitHub Issue。

## Wayfinder 操作

供 `/wayfinder` 使用。地图是一个父议题，工单是其子议题。

- **地图**：使用标签 `wayfinder:map` 的单个议题，用于保存笔记、已有决策和待探索内容。
- **子工单**：与地图建立 GitHub 子议题关系。如果子议题功能不可用，则将子工单加入地图正文的任务列表，并在子工单正文顶部写入 `Part of #<地图编号>`。
- **阻塞关系**：优先使用 GitHub 原生议题依赖关系。
- **前沿查询**：选择地图中第一个仍处于打开状态、没有未关闭阻塞项且尚未分配负责人的子工单。
- **认领**：运行 `gh issue edit <编号> --add-assignee @me`。
- **解决**：使用评论写入处理结果，关闭议题，然后将上下文指针及链接追加到地图的已有决策部分。
