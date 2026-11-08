# NuGet Trusted Publishing

## 发布设计

包仓库先通过 reusable workflow 构建并上传 NuGet artifact，再由包仓库自己的本地 workflow job 发布。发布 job 调用本仓库的 `publish-nuget` Composite Action；该 action 使用 `NuGet/login` 交换短时 API key 并推送到 NuGet.org。

不要在跨仓库 `workflow_call` reusable workflow 中运行 `NuGet/login`。NuGet Trusted Publishing policy 会验证实际调用 workflow 的身份；本地发布 job 调用 Composite Action 可以保留包仓库根 workflow 身份。

仅 NuGet 发布 job 应声明 `id-token: write`。包构建、测试、PR 验证及 Docker job 不应获得该权限。

## GitHub 配置

每个发布仓库需要：

- Secret `NUGET_USER`：NuGet.org 用户资料名，不是邮箱，也不是 API key。可设置为组织级 secret。
- Environment `production`：配置审批者和允许部署的分支/tag。
- 本地发布 job：声明 `permissions.id-token: write` 并调用共享 Composite Action。

## NuGet.org Trusted Publishing Policy

在实际拥有包的 NuGet 用户或组织下创建 policy，按需精确限制 Package ID 和发布权限。每条 policy 绑定 GitHub Owner、Repository、根 workflow 文件名（只填文件名，不含 `.github/workflows/`）及 Environment `production`。

| GitHub repository | Workflow 文件 |
| --- | --- |
| `GameFrameX.Server.Source` | `release.yml` |
| `GameFrameX.SuperSocket` | `release.yml` |
| `GameFrameX.Foundation` | `release.yml` |

组织内统一使用 `release.yml` 作为 NuGet 发布根 workflow。每个仓库仍需在 NuGet.org 单独配置 policy，因为 Repository identity 必须精确匹配；同一仓库内不同触发方式（手动发布与 tag 发布）可由同一个 `release.yml` 处理。确认 NuGet 包的实际 Package Owner 后，再为该 owner 配置对应规则。

## 调用方式

`publish-dotnet-release.yml` 负责版本计算、构建、打包和上传 `nuget-packages` artifact，并输出 `release_needed` 与 `nuget_packages`。调用仓库在本地发布 job 中下载 artifact、申请 OIDC 并发布：

```yaml
  publish-nuget:
    needs: release
    if: ${{ needs.release.outputs.release_needed == 'true' && needs.release.outputs.nuget_packages == 'true' }}
    runs-on: ubuntu-latest
    environment: production
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/download-artifact@v8
        with:
          name: nuget-packages
          path: ./artifacts
      - uses: GameFrameX/public-github-actions/.github/actions/publish-nuget@main
        with:
          package-path: ./artifacts/*.nupkg
          nuget-user: ${{ secrets.NUGET_USER }}
```

tag 触发也应在同一个 `release.yml` 中拆分构建/artifact 上传与本地 OIDC 发布两个 job，避免新增第二个根 workflow policy。

## 迁移验收

1. 在 NuGet.org 为每个根 workflow 配置 Trusted Publisher policy。
2. 在 GitHub 配置 `NUGET_USER` 和 `production` environment。
3. 先发布测试版本，确认 policy 匹配、审批生效且 package scope 正确。
4. 验证成功后再删除旧 `NUGET_API_KEY` secret 并撤销旧 NuGet API key。

迁移期间不要提前撤销仍需回滚的旧 key。MyGet 使用的 `MYGET_API_KEY` 不属于 NuGet.org Trusted Publishing 范围。
