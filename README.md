# public-github-actions

一套开箱即用的 **GitHub Actions 可复用工作流（Reusable Workflows）**集合，用于统一管理组织/个人仓库的发布、镜像同步等常见流程。

> 所有工作流均通过 [`workflow_call`](https://docs.github.com/en/actions/using-workflows/reusing-workflows) 暴露，调用方只需一行 `uses` 即可接入，无需重复编写 CI 脚本。

---

## 🚀 快速开始

1. 在本仓库 **Release** 页面发布一个新版本（或推送 tag），即可自动触发 `release.yml` 生成正式版本。  
2. 在其他仓库中直接引用下方示例，即可立即拥有「自动发布 + 多平台镜像同步」能力。

---

## 📦 可复用工作流一览

| 文件 | 作用 | 入口参数（inputs） | 必需 Secrets / Vars |
|----|------|------------------|---------------------|
| [`.github/workflows/release.yml`](.github/workflows/release.yml) | 自动打 Tag、生成 ChangeLog、创建 GitHub Release | 无（自动读取 `package.json` 中的版本） | `GITHUB_TOKEN`（默认已注入） |
| [`.github/workflows/publish-release.yml`](.github/workflows/publish-release.yml) | 可手动指定 Tag 发布，支持 Unity 项目依赖优化 | `tag_name`, `repository_name` | `GITHUB_TOKEN` |
| [`.github/workflows/sync.yml`](.github/workflows/sync.yml) | **双 Job 同步**：<br>① `sync-to-gitee`（SSH 密钥）<br>② `sync-to-cnb`（GPG 解密令牌 + HTTPS） | `target_branch`, `repository_name`<br>可选 `cnb_repository_name` | **Gitee**（SSH 模式）：<br>`GITEE_ID_RSA`<br>**CNB**（GPG 模式）：<br>`CNB_GPG_PRIVATE_KEY`<br>`CNB_TOKEN_GPG`<br>`CNB_GPG_PASSPHRASE`（可选） |

---

## 🔧 调用示例

### 1. 自动发布 Release

```yaml
name: Release
on:
  push:
    tags: [ 'v*' ]

jobs:
  call:
    uses: your-org/public-github-actions/.github/workflows/release.yml@main
    secrets: inherit   # 自动继承 GITHUB_TOKEN
```

### 2. 同步到 Gitee（SSH 密钥）

```yaml
name: Mirror
on:
  push:
    branches: [ main, develop ]

jobs:
  sync:
    uses: your-org/public-github-actions/.github/workflows/sync.yml@main
    with:
      target_branch: ${{ github.ref_name }}
      repository_name: ${{ github.repository }}
    secrets: inherit   # 需提前在组织/仓库设置 GITEE_ID_RSA
```

### 3. 同步到 CNB（GPG 解密令牌）

```yaml
name: Mirror-CNB
on:
  push:
    branches: [ main ]

jobs:
  sync:
    uses: your-org/public-github-actions/.github/workflows/sync.yml@main
    with:
      target_branch: main
      repository_name: owner/repo
      cnb_repository_name: owner/cnb-repo   # 可省略，默认同 repository_name
    secrets:
      CNB_GPG_PRIVATE_KEY: ${{ secrets.CNB_GPG_PRIVATE_KEY }}
      CNB_TOKEN_GPG:         ${{ secrets.CNB_TOKEN_GPG }}
      CNB_GPG_PASSPHRASE:    ${{ secrets.CNB_GPG_PASSPHRASE }}
```

---

## 🔑 密钥 & 变量准备

### Gitee SSH 模式
1. 生成密钥：`ssh-keygen -t rsa -b 4096 -C "ci@example.com"`
2. 将公钥添加到 Gitee 仓库「部署公钥」
3. 在 GitHub **Settings → Secrets and variables → Actions** 新建 `GITEE_ID_RSA`，粘贴完整私钥内容
4. 在 **Variables** 新建 `GITEE_DOMAIN_URL`（如 `gitee.com`）

### CNB GPG 模式
1. 本地生成 GPG 密钥对：`gpg --full-generate-key`
2. 导出公钥并上传到 CNB 账户「GPG 公钥」
3. 导出私钥：
   ```bash
   gpg --armor --export-secret-keys <key-id> > private.asc
   ```
4. 用公钥加密你的 **Personal Access Token**：
   ```bash
   echo -n '你的Token' | gpg --armor --encrypt -r <key-id> > token.asc
   ```
5. 在 GitHub Secrets 新建：
   - `CNB_GPG_PRIVATE_KEY`：粘贴 `private.asc` 内容
   - `CNB_TOKEN_GPG`：粘贴 `token.asc` 内容
   - `CNB_GPG_PASSPHRASE`（可选）：私钥口令
6. 在 Variables 新建：
   - `CNB_DOMAIN_URL`（如 `codechina.csdn.net`）
   - `CNB_USERNAME`：你的 CNB 用户名

---

## 📚 更多说明

- **ChangeLog 生成逻辑**：基于 [conventional-changelog](https://github.com/conventional-changelog/conventional-changelog)，请使用 [Conventional Commits](https://www.conventionalcommits.org/) 规范提交信息。
- **镜像同步频率**：建议 `on.push` 触发即可，也可改为 `schedule` 定时同步。
- **权限最小化**：所有工作流已声明最小权限集合，调用方无需额外配置。

---

## 🤝 贡献 & 反馈

欢迎提交 Issue 或 PR 来完善这套工作流模板，让更多人「一行 `uses` 搞定发布与同步」！