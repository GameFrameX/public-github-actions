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

## � 工作流实现流程详解

### 1️⃣ publish-release.yml（手动指定 Tag 发布）

| 步骤 | 关键脚本/动作 | 实现要点 |
|----|-------------|---------|
| 检出代码 | `actions/checkout@v4` | `fetch-depth: 0` 保证拿到完整历史，`persist-credentials: false` 强制使用 `GITHUB_TOKEN` 做后续推送，避免权限叠加 |
| 强制拉取标签 | `git fetch --tags --force` | 解决本地与远端标签冲突导致的 "would clobber existing tag" 报错 |
| 安装依赖 | `npm install` | 临时删除 `package.json` 中的 `dependencies` 字段，防止 npm 尝试安装不存在的 Unity 私有依赖；安装完毕再还原文件，保证发布时仍携带依赖声明 |
| 下载共享配置 | `curl -L https://raw.githubusercontent.com/GameFrameX/public-github-actions/main/.releaserc -o .releaserc` | 统一集中管理 `semantic-release` 配置，调用方无需在每个仓库维护 `.releaserc` |
| 语义化发布 | `npx semantic-release` | 按 `.releaserc` 定义执行：<br>① 分析提交 → ② 生成 Release Notes → ③ 打 Tag → ④ 生成/更新 `CHANGELOG.md` → ⑤ 推送到 GitHub Releases → ⑥ 发布到 npm（`CNB_NPM_TOKEN`） |

> 权限声明：`contents: write`（写标签/Release）、`packages: write`（发布 npm）、`checks: write`（状态回写）

---

### 2️⃣ sync.yml（双 Job 镜像同步）

#### Job① sync-to-gitee（SSH 模式）

| 步骤 | 关键脚本/动作 | 实现要点 |
|----|-------------|---------|
| 条件判断 | `if: ${{ secrets.GITEE_ID_RSA != '' }}` | 未提供密钥时整 Job 静默跳过，不影响 CI 结果 |
| 注入 SSH 私钥 | `echo "$GITEE_ID_RSA" > ~/.ssh/id_rsa` | 600 权限 + `ssh-agent` 加载，私钥不落盘 |
| 信任域名 | `ssh-keyscan -H $GITEE_DOMAIN_URL >> ~/.ssh/known_hosts` | 防止首次连接出现 "Are you sure you want to continue connecting" 中断 |
| 添加远端 | `git remote add mirror git@$GITEE_DOMAIN_URL:$repository_name.git` | 使用 SSH 格式，免账号密码 |
| 强制推送 | `git push -f mirror $branch --tags` | 保持与 GitHub 完全一致的分支与标签映射 |

#### Job② sync-to-cnb（GPG + HTTPS 模式）

| 阶段 | 关键脚本/动作 | 实现要点 |
|----|-------------|---------|
| 条件判断 | `if: ${{ secrets.CNB_GPG_PRIVATE_KEY != '' && secrets.CNB_TOKEN_GPG != '' }}` | 两密钥任一缺失即跳过，避免报错 |
| 导入 GPG 私钥 | `gpg --batch --yes --import < $CNB_GPG_PRIVATE_KEY` | 目录权限 700，防止其他用户读取 |
| 解密 Token | `gpg --pinentry-mode loopback --passphrase $CNB_GPG_PASSPHRASE --decrypt token.asc` | 支持「有口令/无口令」两种场景；解密后立刻 `::add-mask::` 隐藏明文 |
| 注入 Basic Auth | `git config --global http.https://$CNB_DOMAIN_URL/.extraheader "AUTHORIZATION: basic $(base64 -w0 $username:$token)"` | 利用 Git 的 `extraheader` 能力，**不把 Token 写入 remote URL**，日志无法泄漏 |
| 动态仓库名 | `cnb_repository_name` 为空时回落到 `repository_name` | 同一套流程可同时支持「同仓镜像」与「异仓镜像」 |
| 强制推送 | 同 Job① | 保持分支与标签完全对齐 |

> 两 Job 并发执行，互不干扰；调用方通过是否提供对应 Secrets 即可「无感」切换目标平台。

---

## �📚 更多说明

- **ChangeLog 生成逻辑**：基于 [conventional-changelog](https://github.com/conventional-changelog/conventional-changelog)，请使用 [Conventional Commits](https://www.conventionalcommits.org/) 规范提交信息。
- **镜像同步频率**：建议 `on.push` 触发即可，也可改为 `schedule` 定时同步。
- **权限最小化**：所有工作流已声明最小权限集合，调用方无需额外配置。

---

## 🤝 贡献 & 反馈

欢迎提交 Issue 或 PR 来完善这套工作流模板，让更多人「一行 `uses` 搞定发布与同步」！