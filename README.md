# Git Operations Skill

`git-operations` 是一个面向 Codex / Agent 的 Git 发布操作技能，用来把中文提交、按功能拆分提交、推送、打标签、合并到 `master`、Jenkins 生产打包、`pnpm build` 静态打包和镜像发布流程约束成可重复、可审计的安全工作流。

这个仓库同时提供两种形态：

- `SKILL.md`：标准 Codex skill 源文件，便于直接阅读和安装。
- `git-operations.skill`：打包后的 skill 文件，内部包含 `git-operations/SKILL.md`。

## 适用场景

当用户要求执行以下任务时，应该使用本 skill：

- `commit`、`提交代码`、`提交并推送`
- `tag`、`tag v1.2.3`、`打标签`、`发版`
- 合并到 `master`
- Jenkins 生产环境打包
- `pnpm build` / `pnpm build:docker` 普通前端打包
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

本 skill 采用“先提交当前改动，再创建发布 tag，再合并到 `master`，最后按项目脚本打包”的模型：

1. 输入 `commit` 时，检查当前工作区，按功能拆分成一个或多个中文提交。
2. 输入 `tag` / `打标签` / `发版` 时，先检查未提交代码；有改动就先走 `commit` 流程，没有可提交代码就直接创建发布 tag。
3. tag 创建并推送后，固定合并到 `master`。
4. 如果需要 Jenkins 生产打包，Jenkins checkout 已打 tag 的提交，校验 tag 后构建并推送镜像。
5. 如果是普通前端项目，按 `package.json` 真实脚本选择 `pnpm build` 或 `pnpm build:docker`。

生产环境打包默认不做 `tag+1`。如果发布提交已经有 tag，就直接使用这个 tag。只有用户输入 `tag` 且未提供版本号时，才根据远端已有 tag 推导下一个版本。

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

| 用户输入                    | 行为                                                          |
| --------------------------- | ------------------------------------------------------------- |
| `tag v1.2.3`                | 检查未提交代码，必要时先中文提交，再使用用户指定版本创建 tag  |
| `tag` / `打标签` / `发版`   | 检查未提交代码，必要时先中文提交，再从远端 tag 推导下一个版本 |
| `Jenkins 打包` / `生产打包` | 使用当前提交已有 tag，不创建新 tag                            |

创建 tag 时的关键规则：

- 先检查未提交改动。
- 必要时按功能拆分中文提交。
- 记录当前分支、当前提交和最终 tag。
- 检查本地和远端是否已有同名 tag。
- 默认创建 annotated tag。
- 默认只推送当前分支和指定 tag，不执行 `git push --tags`。
- tag 推送后固定合并到 `master`。
- 远端已有同名 tag 时停止，不静默 retag。

## 自动推导版本规则

用户输入 `tag`、`打标签`、`发版` 且没有提供版本号时，推导下一个版本。

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
- 当前发布提交已经有 tag，且无法判断是否复用已有 tag
- 无法判断项目版本策略

## 合并到 master

`tag` 流程固定包含合并到 `master`。Jenkins 打包流程不负责合并分支。

推荐策略：

- 输入 `tag` 后，以 `master` 作为固定发布目标分支。
- 合并前确认当前分支、`master`、远端名称、上游状态和 ahead/behind。
- 如果仓库不存在 `master`，停止并询问实际发布分支。
- 本地合并默认使用保守策略，只允许 fast-forward。
- 冲突、目标分支状态不清、需要 merge commit 时停止并询问用户。

推荐顺序：

```bash
SOURCE_BRANCH="$(git branch --show-current)"
RELEASE_TAG="v1.2.3"
git push --atomic origin "$SOURCE_BRANCH" "$RELEASE_TAG"
git switch master
git pull --ff-only origin master
git merge --ff-only "$RELEASE_TAG"
git push origin master
```

如果 `git merge --ff-only "$RELEASE_TAG"` 失败，说明 `master` 不能安全快进到发布 tag，应停止并让用户决定是否通过 PR、merge commit 或其他策略处理。

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

如果项目没有 `build:jenkins` / `build:all`，但有普通前端构建脚本，则按 `package.json` 选择：

```bash
pnpm build
```

如果项目需要输出 Docker/Nginx 静态目录，并且存在 `build:docker`：

```bash
pnpm build:docker
```

例如 `PodifyAdminPc`：

```text
pnpm build        -> vite build，输出 dist/
pnpm build:docker -> vite build --outDir=./docker/dist/，输出 docker/dist/
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

## 普通 pnpm build 打包

不是所有项目都有 `build:all` 或 `build:jenkins`。处理普通前端项目时，必须先读取 `package.json` 的 `scripts`，只运行真实存在的脚本。

推荐选择顺序：

1. 存在 `build:jenkins`：生产镜像流水线优先使用 `pnpm build:jenkins true <registry>`。
2. 存在 `build:all`：服务端镜像全量构建使用 `pnpm build:all <tag> true <registry>`。
3. 普通静态生产包：用户说“普通打包”“静态包”“pnpm build 打包”时使用 `pnpm build`。
4. Docker/Nginx 静态目录：用户说“Docker 打包”“Nginx 部署包”“docker/dist”时，且项目存在 `build:docker`，使用 `pnpm build:docker`。

如果 `build` 和 `build:docker` 同时存在且用户没有说明部署目标，先根据项目 README、Dockerfile、docker 目录和已有产物判断；仍不明确时，给出推荐并询问，不要一次性两个都跑。

即使 README 写的是 `yarn build`，只要项目有 `pnpm-lock.yaml` 且 `package.json` 存在同名 script，也可以优先用 `pnpm` 执行。

## 安全护栏

本 skill 明确禁止 Agent 在未获得用户明确授权时执行以下操作：

- `git reset --hard`
- `git clean -fd`
- `git checkout -- <path>`
- 强推
- 改写历史
- 静默 retag
- 非 `tag` 发布场景自动合并发布分支
- Jenkins 打包时自动 `tag+1`

遇到以下情况必须停止：

- 工作区有来源不明的改动。
- 发现敏感文件或凭证。
- 本地分支落后远端。
- 远端已有同名 tag。
- 当前提交没有发布 tag 却要求生产打包。
- 合并冲突。
- 仓库不存在 `master` 且用户没有指定实际发布分支。
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
docs: 更新 Git 操作技能发布和打包流程
```

推荐发布方式：

1. 提交 `SKILL.md`、`.skill` 包和 `README.md`。
2. 推送到远端。
3. 如需发布仓库版本，在仓库发布提交上创建 tag。
