---
name: git-operations
description: 当用户要求执行 Git 发布操作时使用，包括中文 commit、按功能拆分提交、push、tag、打标签、release、发版、合并到 main/master、Jenkins 生产打包和镜像发布。生产打包流程默认使用 main/master 已有发布 tag，不自动 tag+1；只有用户明确要求创建新 tag 时才推导版本、打 tag 并推送。不要用于纯查询类 Git 请求，例如只查看 status、log、diff 或 blame。
---

# Git Operations

这个技能用于把 Git 提交、推送、打标签、合并发布分支、Jenkins 生产打包做成可重复的安全工作流。它适合用户输入 `commit`、`tag`、`tag v1.2.3`、`打标签`、`发版`、`合并到 main`、`Jenkins 打包`、`生产环境打包` 等场景。

纯查询类请求不要使用本技能，例如用户只想看 `status`、`log`、`diff`、`blame` 或历史记录时，直接按查询处理，不启动提交、tag、合并或打包流程。

## 基本原则

Git 操作会改变共享历史和远端仓库，所以先确认事实，再执行动作。不要凭记忆判断仓库状态，也不要把不相关改动混在一起。

- 先运行只读检查：`git status --porcelain=v1 --branch`、`git diff`、`git diff --staged`、`git log --oneline -10`。
- 检查上游状态：`git log --oneline @{u}..HEAD` 用于查看本地未推送提交，`git log --oneline HEAD..@{u}` 用于查看本地落后远端的提交。
- 没有 upstream 时不要自动发布，先说明拟使用的远端和分支，例如 `git push -u origin <branch>`，等待用户确认。
- 提交前审查将要提交的文件，排除 `.env`、证书、密钥、凭证、临时文件、大文件、异常二进制、生成产物、无关文件和异常子模块变更。
- 不使用 `git reset --hard`、`git clean -fd`、`git checkout -- <path>`、强推、改写历史、静默 retag 或交互式 rebase，除非用户明确要求且再次确认。
- 遇到冲突、未知改动来源、敏感文件、测试失败、构建失败、版本号歧义、远端不可达、权限不足、目标分支状态不清时，停止并报告。

## 发布模型

默认采用“代码先进入发布分支，再用发布分支已有 tag 打生产包”的模型。

- 生产包只来自 `main` 或仓库实际发布分支上的确定提交。
- Jenkins 生产打包默认使用当前提交已有发布 tag，例如 `v1.2.3`。
- Jenkins 打生产包时不要自动 `tag+1`，也不要在打包流水线里自动把 `dev` 合并到 `main`。
- `tag+1` 只用于“用户明确要求创建新发布 tag”的场景；如果 `main` 提交已经有 tag，直接使用这个 tag。
- 部署生产时优先使用精确镜像 tag，例如 `image:v1.2.3`；`latest` 只能作为便捷引用或缓存来源。

推荐职责拆分：

1. 合并流程：通过 PR、人工确认或明确用户指令，把 `dev` 合入 `main`。
2. 打 tag 流程：在 `main` 的发布提交上创建并推送 tag。
3. Jenkins 打包流程：checkout 已打 tag 的提交，校验 tag，构建并推送镜像。

## 用户输入 `commit` 时

当用户只输入 `commit` 或要求“提交代码”“提交并推送”时，执行以下流程：

1. 检查仓库状态、未暂存差异、已暂存差异、未跟踪文件和最近提交。
2. 如果没有可提交内容，报告无需提交，不创建空提交。
3. 按功能理解改动，把相关文件归成一组。一个功能、修复、配置调整或文档更新对应一个提交。
4. 如果改动明显属于多个功能，可以多次提交。每次提交前只暂存该提交需要的文件。
5. 不要把格式化、重构、业务逻辑、构建配置和文档改动无理由混在同一个提交里。
6. 提交信息使用中文，简短说明这次提交的业务意图。
7. 提交前再次查看 `git diff --staged`，确认暂存内容正确。
8. 默认不使用 `git commit --amend`；只有用户明确要求修改最近一次提交并确认风险时才考虑。
9. 如果用户要求推送，按“推送规则”执行。

提交信息示例：

```text
修复登录态过期提示
新增订单导出入口
调整前端打包配置
补充接口异常处理
```

## 推送规则

推送前确认当前分支、上游分支、远端名称、ahead/behind 状态。

- 本地落后远端时停止，先报告远端新增提交，不自动 merge 或 rebase。
- 本地领先远端且用户要求 push 时，只推送当前分支。
- 无 upstream 时，先展示拟执行命令，例如 `git push -u origin <branch>`，等待用户确认。
- 远端不可达、权限不足、分支不存在或推送被拒绝时停止，不尝试高风险修复。

## 用户输入 `tag` 或 `发版` 时

先判断用户要的是“创建新发布 tag”还是“用已有 tag 打生产包”。不要把两者混在一起。

- 用户说 `tag v1.2.3`、`打标签 1.4.0`：创建用户指定版本的发布 tag。
- 用户只说 `tag`、`打标签`、`发版` 且没有版本号：可以推导下一个版本，但必须先确认这是创建新 tag，不是 Jenkins 使用已有 tag 打包。
- 用户说 `Jenkins 打包`、`生产打包`、`用 main 的 tag 打包`：按“Jenkins 生产打包”流程，不自动创建新 tag。

创建新发布 tag 的流程如下：

1. 检查本地是否有未提交代码、未暂存改动、已暂存改动和未跟踪文件。
2. 如果存在未提交改动，按 `commit` 工作流拆分功能并使用中文提交，可以根据功能提交多次。
3. 检查上游状态。如果存在本地未推送提交，按“推送规则”先 push 到远端；如果本地落后远端，停止并报告。
4. 确定 tag 版本号：带版本号时使用用户提供版本；未提供版本号时自动推导下一个版本并向用户说明。
5. 确认本地和远端都不存在同名 tag。远端已有同名 tag 时停止，不删除重建、不移动、不强推覆盖。
6. 创建发布 tag。默认使用 annotated tag：`git tag -a <version> -m <version>`；如果仓库已经配置签名且签名可用，优先使用 signed tag：`git tag -s <version> -m <version>`。
7. push 到远端。默认只推送指定 tag：`git push origin <version>`，不要使用 `git push --tags`。如果需要同时推送分支和 tag，优先使用 `git push --atomic origin <branch> <version>`。
8. 如果用户明确要求合并到发布分支，按“合并到 main/master”流程执行；不要因为 tag 流程而自动合并分支。
9. 如果用户明确要求本地打包，使用项目既有打包脚本；生产镜像优先交给 Jenkins。
10. 输出结果摘要，流程结束。

## 自动推导下一个版本

未提供版本号且用户确认要创建新 tag 时，先读取远端 tag，再推导下一个版本，不只依赖本地 tag。

1. 先执行：`git fetch origin --tags --prune --prune-tags`。
2. 再按版本排序查看候选 tag，例如：`git tag --list 'v*' --sort=-version:refname`。
3. 仅自动处理简单版本：`v1.2.3` 变为 `v1.2.4`，`1.2.3` 变为 `1.2.4`，`v7` 变为 `v8`。
4. 遇到预发布版本、日期版本、混合前缀、多条版本线、格式混乱、没有可判断版本或项目使用特殊版本策略时，停止并询问用户版本号。
5. 如果当前发布分支提交已经有发布 tag，不要再推导 `tag+1`，除非用户明确要创建新的发布点。

## 合并到 main/master

合并分支是独立流程，不属于 Jenkins 生产打包步骤。只有用户明确要求“合并到 main/master”时才执行。

1. 确认当前分支、目标分支、远端名称、上游分支和 ahead/behind 状态。
2. 优先识别仓库默认发布分支：存在 `main` 时优先用 `main`；只有仓库实际使用 `master` 或用户明确要求时才用 `master`。
3. 获取远端最新状态，确保源分支和目标分支都不是过期状态。
4. 默认推荐通过 PR 或代码平台合并；如果用户要求本地合并，先说明将执行的源分支和目标分支。
5. 默认使用保守合并策略，优先 fast-forward，例如 `git merge --ff-only <source>`。
6. 如果需要 merge commit、发生冲突、目标分支状态不清或合并策略不明确，停止并询问用户。
7. 合并成功后只推送目标分支，不推送无关分支。

## Jenkins 生产打包

当用户要求 Jenkins 打生产环境包时，默认使用“当前提交已有 tag”模式。

固定流程：

1. 确认代码已经合入 `main` 或仓库实际发布分支。
2. 确认发布提交已经存在 tag，例如 `v1.2.3`。
3. Jenkins checkout 精确 tag；如果只能 checkout 分支，则必须校验当前 `HEAD` 上有发布 tag。
4. 拉取远端 tag：`git fetch --tags --force origin`。
5. 校验构建版本：优先运行项目提供的 `node scripts/build-jenkins.mjs --print-build-tag`；没有该脚本时，用 `git tag --points-at HEAD` 检查当前提交 tag。
6. 当前提交没有 tag、存在多个无法判断的 tag、tag 不符合项目版本规则时，停止打包。
7. 登录镜像仓库后运行项目的 Jenkins 打包入口。若仓库提供 `pnpm build:jenkins`，使用：`pnpm build:jenkins true <registry>`。
8. 如果没有 `pnpm build:jenkins`，才退回项目既有脚本：`pnpm build:all <tag> true <registry>`。
9. 打包成功后报告镜像 tag、构建提交、Jenkins build number 和推送结果。

推荐 Jenkinsfile 片段：

```groovy
pipeline {
  agent any
  options {
    timestamps()
    disableConcurrentBuilds()
  }
  environment {
    CI = 'true'
    DOCKER_IMAGE = 'registry.example.com/group/app'
    DOCKER_REGISTRY_HOST = 'registry.example.com'
  }
  stages {
    stage('检出代码') {
      steps {
        checkout scm
        sh 'git fetch --tags --force origin'
      }
    }
    stage('确认发布标签') {
      steps {
        sh 'node scripts/build-jenkins.mjs --print-build-tag'
      }
    }
    stage('登录镜像仓库') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'docker-registry', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
          sh 'echo "$DOCKER_PASSWORD" | docker login --username="$DOCKER_USERNAME" --password-stdin "$DOCKER_REGISTRY_HOST"'
        }
      }
    }
    stage('生产打包并推送') {
      steps {
        sh 'pnpm build:jenkins true "$DOCKER_IMAGE"'
      }
    }
  }
}
```

## 本地打包

用户单独要求本地构建或打包时，按项目现有脚本判断，不替用户切换发布策略。

- 有明确版本号时：`pnpm build:all <version> true` 或项目指定命令。
- 没有明确版本号但当前提交有 tag 时：优先使用当前提交 tag。
- 没有 tag 时不要自动 `tag+1` 打生产包，除非用户明确要创建新 tag。
- 打包失败时停止，报告失败命令和关键错误。

## 禁止静默 retag

已发布 tag 是共享发布记录，不要悄悄改写。

- 远端已有同名 tag 时，停止并报告本地 tag、远端 tag 和当前目标提交的差异。
- 不删除、不重建、不移动、不强推覆盖已发布 tag。
- 如果用户明确要求修正错误 tag，先说明影响，再等待用户确认具体处理方式。

## 输出格式

执行前给出简短计划，执行后给出结果摘要。字段保持稳定，便于用户检查。

```markdown
**计划**

- 当前分支：...
- 上游分支：...
- 工作区状态：...
- 计划提交数量：...
- 准备推送：...
- tag 版本：...
- 发布分支：main/master/无
- Jenkins 打包：是/否
- 打包命令：...
- 阻塞风险：...

**结果**

- 已提交：...
- 已推送分支：...
- 已创建 tag：...
- 已推送 tag：...
- 已合并发布分支：...
- Jenkins 打包版本：...
- 打包结果：...
- 阻塞原因：...
```

如果任何步骤被阻塞，用 `[blocked]` 标出原因和用户需要决定的事项。

## 验收场景

- 用户输入 `commit`，工作区干净：报告无需提交，不创建空提交。
- 用户输入 `commit`，有多个功能改动：按功能拆分多个中文提交。
- 用户输入 `tag v1.2.3`：使用 `v1.2.3`，检查未提交代码，必要时中文提交，创建发布 tag，push 到远端；不自动合并或打包，除非用户明确要求。
- 用户输入 `tag`：先确认是创建新发布 tag，再读取远端 tag 并推导下一个版本。
- 用户要求 Jenkins 打生产包：checkout 已有 tag 或校验当前提交已有 tag，运行 `pnpm build:jenkins true <registry>`，不自动 `tag+1`。
- 当前提交没有 tag 却要求生产打包：停止，要求先在发布提交上创建 tag。
- 远端已有同名 tag：停止，不静默 retag。
- 没有 upstream：展示 `git push -u origin <branch>`，等待确认。
- 默认发布分支不是 `main` 或 `master`：询问用户实际发布分支。
- 合并冲突、权限不足或打包失败：停止并报告。

## 典型触发

- 用户输入：`commit`
  - 检查状态和 diff，按功能拆分中文提交；如果用户要求 push，则按推送规则执行。
- 用户输入：`tag`
  - 确认是否创建新发布 tag；确认后自动推导下一个版本，中文提交未提交代码，push 指定 tag。
- 用户输入：`tag v2.3.0`
  - 使用 `v2.3.0`，中文提交未提交代码，创建并 push 指定 tag。
- 用户输入：`Jenkins 打生产包`
  - 使用当前 `main`/发布分支提交已有 tag，运行 Jenkins 打包命令，不创建新 tag。
