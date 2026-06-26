# Git Operations Skill

`git-operations` 是一个面向 Codex / OpenCode Agent 的 Git 发布操作技能，用来把提交、推送、打标签、发布分支合并、Jenkins 生产打包和镜像发布流程约束成可重复、可审计的安全工作流。

这个仓库同时提供两种形态：

- `SKILL.md`：标准 Codex skill 源文件，便于直接阅读和安装。
- `git-operations.skill`：打包后的 skill 文件，内部包含 `git-operations/SKILL.md`。

## 适用场景

当用户要求执行以下任务时，应该使用本 skill：

- `commit`、`提交代码`、`提交并推送`
- `tag`、`tag v1.2.3`、`打标签`、`发版`
- 合并到 `main` / `master`
- Jenkins 生产环境打包
- Docker 镜像发布
- Git 发布流程风险检查

当用户只是查看信息时，不应该触发本 skill，例如：

- `git status`
- `git log`
- `git diff`
- `git blame`
- 查看某个历史提交

这类纯查询请求直接执行查询即可，不启动提交、tag、合并或打包流程。

## 核心发布模型

本 skill 采用“先发布分支，再发布 tag，最后 Jenkins 打包”的模型：

1. 开发分支通过 PR、代码平台或明确用户指令合入 `main` / `master`。
2. 在发布分支的确定提交上创建发布 tag，例如 `v1.2.3`。
3. Jenkins checkout 已打 tag 的提交。
4. Jenkins 校验当前提交确实有发布 tag。
5. Jenkins 使用该 tag 作为 Docker 镜像版本构建并推送。

生产环境打包默认不做 `tag+1`。如果 `main` 当前提交已经有 tag，就直接使用这个 tag。只有用户明确要求“创建新发布 tag”时，才允许根据远端已有 tag 推导下一个版本。

## 为什么生产打包不自动 tag+1

生产镜像应该对应一个已经确认的发布提交。自动 `tag+1` 容易造成以下问题：

- Jenkins 在错误提交上创建新 tag。
- 镜像版本和代码评审记录不一致。
- 回滚时无法清楚判断哪个提交对应哪个镜像。
- 并发流水线可能计算出相同的下一个版本。

因此推荐规则是：

- 创建 tag 是发布动作。
- 使用 tag 打包是构建动作。
- 合并分支是独立动作。
- Jenkins 生产打包只消费已有 tag，不自动制造发布版本。

## 安装方式

### 安装到 Codex

将仓库中的 `SKILL.md` 放到本地 Codex skill 目录：

```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.codex\skills\git-operations"
Copy-Item .\SKILL.md "$env:USERPROFILE\.codex\skills\git-operations\SKILL.md" -Force
```

安装后重启或刷新 Codex，让 skill 被重新发现。

### 使用 `.skill` 包

`git-operations.skill` 是 zip 格式的打包文件，内部结构如下：

```text
git-operations/
└── SKILL.md
```

如果你的工具支持导入 `.skill` 文件，可以直接导入该文件。

## 使用方式

在 Codex 或支持 skill 的 Agent 中，用自然语言提出 Git 发布任务即可。

示例：

```text
commit
```

```text
提交并推送
```

```text
tag v1.2.3
```

```text
帮我发版
```

```text
用 Jenkins 打生产环境包
```

## Commit 工作流

当用户要求提交代码时，skill 会要求 Agent：

1. 先读取仓库状态、未暂存 diff、已暂存 diff、最近提交。
2. 判断是否有可提交内容。
3. 按功能拆分提交，不把无关改动混入同一提交。
4. 排除敏感文件、临时文件、生成产物和异常二进制。
5. 使用中文提交信息。
6. 如果用户要求推送，再按推送规则处理。

提交信息示例：

```text
修复登录态过期提示
新增订单导出入口
调整前端打包配置
补充接口异常处理
```

## Tag / 发版工作流

当用户要求创建发布 tag 时，skill 会区分两种情况：

| 用户输入                    | 行为                                              |
| --------------------------- | ------------------------------------------------- |
| `tag v1.2.3`                | 使用用户指定版本创建 tag                          |
| `tag` / `打标签` / `发版`   | 先确认是否创建新 tag，再从远端 tag 推导下一个版本 |
| `Jenkins 打包` / `生产打包` | 使用当前提交已有 tag，不创建新 tag                |

创建 tag 时的关键规则：

- 先检查未提交改动。
- 必要时按功能拆分中文提交。
- 检查本地和远端是否已有同名 tag。
- 默认创建 annotated tag。
- 默认只推送指定 tag，不执行 `git push --tags`。
- 远端已有同名 tag 时停止，不静默 retag。

## 自动推导版本规则

只有用户明确要创建新 tag，且没有提供版本号时，才推导下一个版本。

推荐读取远端 tag：

```bash
git fetch origin --tags --prune --prune-tags
git tag --list 'v*' --sort=-version:refname
```

支持的简单规则：

```text
v1.2.3 -> v1.2.4
1.2.3  -> 1.2.4
v7     -> v8
```

遇到以下情况必须停止并询问用户：

- 预发布版本，例如 `v1.2.3-beta.1`
- 日期版本
- 多条版本线
- tag 前缀混乱
- 当前发布提交已经有 tag
- 无法判断项目版本策略

## 合并到 main / master

合并分支是独立发布动作，不属于 Jenkins 打包步骤。

推荐策略：

- 优先通过 PR 或代码平台合并。
- 仓库存在 `main` 时优先认为 `main` 是发布分支。
- 只有仓库实际使用 `master` 或用户明确要求时，才使用 `master`。
- 本地合并默认使用保守策略，例如 `git merge --ff-only`。
- 冲突、目标分支状态不清、需要 merge commit 时停止并询问用户。

## Jenkins 生产打包流程

推荐 Jenkins 流程：

1. checkout 发布 tag，或 checkout 发布分支后校验当前提交已有 tag。
2. 拉取远端 tag。
3. 校验当前提交上的发布 tag。
4. 登录 Docker 镜像仓库。
5. 使用发布 tag 构建并推送镜像。

推荐命令：

```bash
git fetch --tags --force origin
node scripts/build-jenkins.mjs --print-build-tag
pnpm build:jenkins true "$DOCKER_IMAGE"
```

如果项目没有 `pnpm build:jenkins`，可以退回既有脚本：

```bash
TAG="$(git tag --points-at HEAD | head -n 1)"
pnpm build:all "$TAG" true "$DOCKER_IMAGE"
```

## Jenkinsfile 示例

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
        withCredentials([usernamePassword(
          credentialsId: 'docker-registry',
          usernameVariable: 'DOCKER_USERNAME',
          passwordVariable: 'DOCKER_PASSWORD'
        )]) {
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

## 安全护栏

本 skill 明确禁止 Agent 在未获得用户明确授权时执行以下操作：

- `git reset --hard`
- `git clean -fd`
- `git checkout -- <path>`
- 强推
- 改写历史
- 静默 retag
- 自动合并发布分支
- Jenkins 打包时自动 `tag+1`

遇到以下情况必须停止：

- 工作区有来源不明的改动。
- 发现敏感文件或凭证。
- 本地分支落后远端。
- 远端已有同名 tag。
- 当前提交没有发布 tag 却要求生产打包。
- 合并冲突。
- 权限不足。
- 测试或构建失败。

## 维护方式

更新 skill 时建议同步维护以下文件：

```text
SKILL.md
git-operations.skill
README.md
```

重新生成 `.skill` 包时，保持内部目录结构：

```text
git-operations/SKILL.md
```

在 PowerShell 中可以使用：

```powershell
$temp = Join-Path $PWD '.skill-package'
Remove-Item $temp -Recurse -Force -ErrorAction SilentlyContinue
New-Item -ItemType Directory -Force (Join-Path $temp 'git-operations')
Copy-Item .\SKILL.md (Join-Path $temp 'git-operations\SKILL.md') -Force
Compress-Archive -Path (Join-Path $temp 'git-operations') -DestinationPath .\git-operations.skill -Force
Remove-Item $temp -Recurse -Force
```

## 校验建议

更新后建议至少检查：

```bash
git status --porcelain=v1 --branch
git diff -- SKILL.md README.md
```

如果有 Codex `skill-creator` 的校验脚本：

```bash
python quick_validate.py <skill-folder>
```

也可以手动检查：

- `SKILL.md` 是否有合法 YAML frontmatter。
- `name` 是否为 `git-operations`。
- `description` 是否明确包含触发场景。
- Jenkins 章节是否明确“不自动 tag+1”。

## 发布建议

推荐提交信息：

```text
docs: 更新 Git 操作技能 Jenkins 发布流程
```

推荐发布方式：

1. 提交 `SKILL.md`、`.skill` 包和 `README.md`。
2. 推送到远端。
3. 如需发布仓库版本，在仓库发布提交上创建 tag。
