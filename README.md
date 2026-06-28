# public-github-actions

一套给 GameFrameX 仓库复用的 **GitHub Actions 工作流集合**，用于统一处理发布、历史标签补发、.NET 发布，以及从 GitHub 同步到 Gitee 和 CNB.Cool。

所有工作流都通过 `workflow_call` 暴露。业务仓库只需要保留一层很薄的调用工作流，实际逻辑集中维护在本仓库。

## 快速接入

在业务仓库的 `.github/workflows/` 中添加调用文件，例如镜像同步：

```yaml
name: Sync Github To Image (External)

on:
  push:
    branches:
      - '**'
    tags-ignore:
      - '**'
  workflow_run:
    workflows: ["Publish Release"]
    types:
      - completed
  workflow_dispatch:

jobs:
  sync-gitee:
    uses: GameFrameX/public-github-actions/.github/workflows/sync.yml@main
    with:
      target_branch: ${{ github.ref_name }}
      repository_name: ${{ github.repository }}
    secrets: inherit
```

## 可复用工作流

| 文件 | 作用 | Inputs | Secrets / Vars |
| --- | --- | --- | --- |
| [`.github/workflows/publish-release.yml`](.github/workflows/publish-release.yml) | Unity/通用仓库发布：按传入分支或 tag 执行 semantic-release，生成 changelog、tag、GitHub Release，并可发布到 CNB npm | `tag_name`, `repository_name` | `GITHUB_TOKEN`, `CNB_NPM_TOKEN` |
| [`.github/workflows/publish-historical-tags.yml`](.github/workflows/publish-historical-tags.yml) | 手动补发历史 tag 的 Release | `tags` | `GITHUB_TOKEN` |
| [`.github/workflows/sync.yml`](.github/workflows/sync.yml) | 双 Job 镜像同步：自动读取 GitHub 仓库元数据，按需创建 Gitee/CNB 仓库，同步描述、站点、主题，并推送代码和 tags | `target_branch`, `repository_name` | `GITHUB_TOKEN`; Gitee: `GITEE_GITHUB_ACTION_SYNC_TOKEN`, `GITEE_ID_RSA`; CNB: `CNB_SYNC_TOKEN`; Vars: `GIT_USER_EMAIL`, `GIT_USER_NAME`, `GITEE_DOMAIN_URL` |
| [`.github/workflows/publish-dotnet-release.yml`](.github/workflows/publish-dotnet-release.yml) | .NET 语义化发布：计算版本、生成 changelog、可选更新 Version.props、发布 NuGet、构建并推送 Docker 镜像、创建 GitHub Release | `dotnet_version`, `version_props_path`, `nuget_source`, `docker_image_name`, `docker_platforms`, `docker_context` | `GITHUB_TOKEN`; NuGet: `NUGET_API_KEY`; Docker: `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`, `DOCKERHUB_ORGANIZATION` |

## Sync 工作流行为

`sync.yml` 是当前镜像体系的主流程。Gitee 和 CNB 的仓库创建、描述信息同步、代码同步都优先由这个工作流完成。

### 读取 GitHub 元数据

两个同步 Job 都会先通过 `gh repo view` 读取主仓库信息：

- `description`
- `homepageUrl`
- `repositoryTopics`

这些信息作为 Gitee 和 CNB 的同步源。也就是说，仓库描述、网站和 topics 应优先在 GitHub 主仓库维护，再由同步工作流分发到镜像平台。

### Gitee 同步

`sync-to-gitee` 会执行以下动作：

1. 使用 `GITEE_GITHUB_ACTION_SYNC_TOKEN` 查询 `GameFrameX/{repo}` 是否存在。
2. 仓库不存在时，通过 Gitee API 创建公开仓库。
3. 仓库已存在时，比较 Gitee 的 `description`、`homepage` 与 GitHub 是否一致。
4. 新建仓库或信息不一致时，通过 PATCH API 更新描述和网站。
5. 使用 `GITEE_ID_RSA` 配置 SSH。
6. 添加远端 `git@${GITEE_DOMAIN_URL}:${repository_name}.git`。
7. 强制推送当前分支，并同步 tags。

### CNB 同步

`sync-to-cnb` 会执行以下动作：

1. 使用 `CNB_SYNC_TOKEN` 查询 `gameframex/{repo}` 是否存在。
2. 仓库不存在时，通过 CNB API 创建公开仓库。
3. 仓库已存在时，比较 CNB 的 `description`、`site`、`topics` 与 GitHub 是否一致。
4. 新建仓库或信息不一致时，通过 PATCH API 更新描述、站点和 topics。
5. topics 会过滤为长度不超过 12 的项目，并最多保留 12 个。
6. 使用 `docker://tencentcom/git-sync` 同步到 `https://cnb.cool/${repository_name}.git`。
7. 同步模式为 `rebase`，启用强制同步和 tags 推送。

## Secrets 和 Variables

### GitHub

`GITHUB_TOKEN` 由 GitHub Actions 默认注入。调用方使用 `secrets: inherit` 即可。

### Gitee

需要在组织或仓库配置：

- Secret: `GITEE_GITHUB_ACTION_SYNC_TOKEN`
  - 用于查询、创建、更新 Gitee 仓库信息。
- Secret: `GITEE_ID_RSA`
  - 用于 SSH 推送代码到 Gitee。
- Variable: `GITEE_DOMAIN_URL`
  - 通常为 `gitee.com`。
- Variable: `GIT_USER_EMAIL`
  - 用于 Git 提交/推送环境配置。
- Variable: `GIT_USER_NAME`
  - 用于 Git 提交/推送环境配置。

### CNB

需要在组织或仓库配置：

- Secret: `CNB_SYNC_TOKEN`
  - 用于查询、创建、更新 CNB 仓库，并作为 `git-sync` 的 HTTPS token。

### 发布

按需配置：

- `CNB_NPM_TOKEN`
- `NUGET_API_KEY`
- `DOCKERHUB_USERNAME`
- `DOCKERHUB_TOKEN`
- `DOCKERHUB_ORGANIZATION`

## 调用示例

### 发布 Release

```yaml
name: Publish Release

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  publish-release:
    uses: GameFrameX/public-github-actions/.github/workflows/publish-release.yml@main
    with:
      tag_name: ${{ github.ref_name }}
      repository_name: ${{ github.repository }}
    secrets: inherit
```

### 补发历史 Tags

```yaml
name: Publish Historical Tags

on:
  workflow_dispatch:

jobs:
  publish-historical-tags:
    uses: GameFrameX/public-github-actions/.github/workflows/publish-historical-tags.yml@main
    with:
      tags: ''
    secrets: inherit
```

### .NET 包发布

```yaml
name: Release

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  release:
    uses: GameFrameX/public-github-actions/.github/workflows/publish-dotnet-release.yml@main
    with:
      dotnet_version: "10.0.x"
      version_props_path: "Version.props"
    secrets: inherit
```

### Docker 镜像发布

```yaml
name: Release

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  release:
    uses: GameFrameX/public-github-actions/.github/workflows/publish-dotnet-release.yml@main
    with:
      dotnet_version: "10.0.x"
      docker_image_name: "gameframex-tools"
    secrets: inherit
```

## 主流程和兜底流程

GameFrameX 仓库的推荐流程是：

1. 在 GitHub 主仓库维护 description、homepage 和 topics。
2. 业务仓库引用本仓库的 `sync.yml`。
3. 由 `sync.yml` 自动创建或更新 Gitee/CNB 镜像仓库。
4. 仅当 Actions 无法完成自动创建、平台 API 异常、token 权限异常、历史仓库状态不一致时，再使用本地 Skill 做手动修复。

因此，Gitee/CNB 平台侧的独立 Skill 不再属于常规主流程能力，而是故障修复和一次性兜底工具。

## 维护说明

- 修改同步逻辑时，优先修改 `.github/workflows/sync.yml`，再同步更新本 README。
- 修改发布逻辑时，保持业务仓库模板只做薄调用，避免把实现逻辑复制到每个仓库。
- 镜像仓库的元数据源头是 GitHub 主仓库；不要在 Gitee/CNB 长期维护不同的描述信息。
