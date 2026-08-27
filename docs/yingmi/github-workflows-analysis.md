# GitHub Actions Workflows 分析

本文档基于仓库当前 `.github/workflows/*.yml` 的配置整理，覆盖 27 个
workflow（分析日期：2026-08-27）。`.github/workflows/README.md`、
`Dockerfile.bazel` 和 `.github/actions/` 是 workflow 使用的辅助文件，不是
独立 workflow；文末另有说明。

## 快速总览（按重要程度排列）

下表把合并、发布和自动化入口排在前面，随后列出 reusable workflow。`触发次数`
是 GitHub Actions workflow 历史页显示的累计值；GitHub 对高频 workflow 截断为
`2,500+`。`典型运行时间`取近期历史页中可见的已完成运行作参考，不代表 SLA，也
没有把排队时间单独剔除。reusable workflow 被调用时作为 caller 的子 job 执行，
GitHub 通常不会在其独立页面累计一条同等意义的 workflow run，因此次数/耗时以
“随调用方统计”表示。

| 重要度 | Workflow | 用途 | 触发机制 | 定时任务 | Runner 类型 | 触发次数（历史页） | 典型运行时间 | 被哪个 workflow 调用 |
| --- | --- | --- | --- | --- | --- | ---: | --- | --- |
| P0 | `blocking-ci.yml` | PR 合并阻断总入口 | `pull_request`；push `main` | 否 | 子 workflow 各自使用 Linux/macOS/Windows | 2,500+ | 8–31 分钟 | —（入口） |
| P0 | `rust-release.yml` | Codex Rust 多平台构建、签名、Release 和分发 | push `rust-v*.*.*` tag | 否 | Ubuntu、macOS x64/arm64、Windows 自托管 runner | 1,297 | alpha 约 7 分钟；完整稳定版约 50–62 分钟 | —（入口） |
| P0 | `postmerge-ci.yml` | `main` 合并后的完整验证 | push `main` | 否 | Ubuntu、macOS、Windows（由子 workflow 决定） | 1,730 | 约 9–13 分钟 | —（入口） |
| P0 | `bazel.yml` | Bazel test、clippy、release build 验证 | `workflow_call`；手动运行 | 否 | Ubuntu 24.04、macOS 15、Windows | 2,500+ | 约 2–31 分钟 | `blocking-ci.yml` |
| P0 | `rust-ci.yml` | PR 快速 Rust/Cargo 检查与变更检测 | `workflow_call`；手动运行 | 否 | Ubuntu；macOS；Windows 自托管 runner | 2,500+ | 约 7–31 分钟 | `blocking-ci.yml` |
| P0 | `rust-ci-full.yml` | 完整 Cargo clippy、nextest、多平台测试 | `workflow_call`；`**full-ci**` 分支 push；手动运行 | 否 | Ubuntu x64/arm64、macOS、Windows x64/arm64 | 2,500+ | 约 9–41 分钟 | `postmerge-ci.yml` |
| P0 | `v8-canary.yml` | 检测并构建 V8 canary 变更 | `workflow_call`；`pull_request`；手动运行 | 否 | Ubuntu、macOS、Windows | 2,500+ | 约 1–30 分钟 | `postmerge-ci.yml` |
| P1 | `python-sdk-release.yml` | 发布 Python runtime 和 Python SDK 到 PyPI | push `python-v*` tag | 否 | Ubuntu | 9 | 约 4 分钟 | —（入口） |
| P1 | `rusty-v8-release.yml` | 发布 rusty_v8 预编译产物和 Windows source 产物 | push `rusty-v8-v*.*.*` tag | 否 | Ubuntu x64/arm64、macOS x64/arm64、Windows | 12 | 约 5 分钟至 2 小时 44 分钟 | —（入口） |
| P1 | `rust-release-prepare.yml` | 定期更新 `models.json` 并创建 PR | 手动运行；cron `0 */4 * * *` | 是，每 4 小时 | Ubuntu | 1,554 | 约 40 秒–4 分钟 | —（入口） |
| P1 | `issue-deduplicator.yml` | 用 Codex 检测重复 Issue 并评论 | Issue `opened`；加 `codex-deduplicate` label | 否 | Ubuntu | 2,500+ | 约 2 秒–2 分 16 秒 | —（入口） |
| P1 | `issue-translator.yml` | 翻译新建的非英文 Issue | Issue `opened` | 否 | Ubuntu | 2,500+ | 约 39–45 秒 | —（入口） |
| P1 | `issue-labeler.yml` | 用 Codex 生成并应用 Issue 标签建议 | Issue `opened`；加 `codex-label` label | 否 | Ubuntu | 2,500+ | 约 1–45 秒 | —（入口） |
| P1 | `cla.yml` | 检查贡献者 CLA 并更新 PR 状态 | `issue_comment.created`；PR `opened/closed/synchronize` | 否 | Ubuntu | 2,500+ | 约 6–11 秒 | —（入口） |
| P1 | `close-stale-contributor-prs.yml` | 关闭 14 天不活跃的 contributor PR | 手动运行；cron `0 6 * * *` | 是，每天 | Ubuntu | 217 | 约 20 秒–1 分 30 秒 | —（入口） |
| P1 | `sdk.yml` | 测试 Python SDK 和 TypeScript SDK | `workflow_call` | 否 | Ubuntu glibc image；Linux x64 runner | 2,500+ | 约 10 分钟 | `blocking-ci.yml` |
| P1 | `repo-checks.yml` | 仓库结构、格式、installer/package 和文档检查 | `workflow_call` | 否 | Ubuntu | 0（独立历史页） | 预计 1–10 分钟；随 caller 统计 | `blocking-ci.yml` |
| P1 | `blob-size-policy.yml` | 检查变更 blob 大小策略 | `workflow_call` | 否 | Ubuntu 24.04 | 2,500+ | 约 40 秒 | `blocking-ci.yml` |
| P1 | `cargo-deny.yml` | 检查 Rust 依赖许可和安全策略 | `workflow_call` | 否 | Ubuntu | 2,500+ | 约 1–2 分钟 | `blocking-ci.yml` |
| P1 | `codespell.yml` | 检查拼写错误 | `workflow_call` | 否 | Ubuntu | 2,500+ | 约 30–40 秒 | `blocking-ci.yml` |
| P1 | `python-runtime-release.yml` | 手动发布指定 Python runtime 版本 | 手动运行，必填 `runtime_version` | 否 | Ubuntu | 页面未稳定返回计数 | 预计 3–6 分钟；随 caller 统计 | —（入口） |
| P2 | `rust-release-windows.yml` | Windows 二进制、symbols、签名和 wheel | `workflow_call` | 否 | Windows x64/arm64 自托管 runner；Ubuntu 汇总 | 随调用方统计 | 通常 20–90 分钟 | `rust-release.yml` |
| P2 | `rust-release-zsh.yml` | 构建和发布 zsh wrapper 产物 | push `codex-zsh-v*.*.*` tag | 否 | Ubuntu x64/arm64、macOS x64/arm64 | 页面未稳定返回计数 | 预计 3–15 分钟；随发布流程变化 | —（入口） |
| P2 | `r2-release.yml` | 把 release assets/metadata 发布到 Cloudflare R2 | `workflow_call`，由 caller 传入 tag/stage 等 | 否 | Ubuntu | 随调用方统计 | 通常 1–5 分钟 | `rust-release.yml` |
| P2 | `rust-release-argument-comment-lint.yml` | 构建正式版 argument-comment-lint release assets | `workflow_call`，input `publish` | 否 | Ubuntu、Ubuntu arm64、macOS、Windows | 随调用方统计 | 通常 5–20 分钟 | `rust-release.yml` |
| P2 | `rust-ci-full-nextest-platform.yml` | 生成 nextest archive，并分 shard 运行平台测试 | `workflow_call`，由完整 Rust CI 传入 target/profile | 否 | caller 指定的 Ubuntu/macOS/Windows runner | 随调用方统计 | 通常 10–40 分钟 | `rust-ci-full.yml` |
| P2 | `python-runtime-build.yml` | 构建并上传 musllinux Python runtime wheels | `workflow_call`，input `runtime_version` | 否 | Ubuntu | 随调用方统计 | 通常 2–5 分钟 | `python-runtime-release.yml`、`python-sdk-release.yml` |

历史数据来源：GitHub 的 [Actions 总览](https://github.com/openai/codex/actions)、
[blocking-ci 历史](https://github.com/openai/codex/actions/workflows/blocking-ci.yml)、
[postmerge-ci 历史](https://github.com/openai/codex/actions/workflows/postmerge-ci.yml)、
[rust-ci 历史](https://github.com/openai/codex/actions/workflows/rust-ci.yml)、
[rust-ci-full 历史](https://github.com/openai/codex/actions/workflows/rust-ci-full.yml)、
[rust-release 历史](https://github.com/openai/codex/actions/workflows/rust-release.yml)、
[Bazel 历史](https://github.com/openai/codex/actions/workflows/bazel.yml)、
[Python SDK release 历史](https://github.com/openai/codex/actions/workflows/python-sdk-release.yml)、
[rusty_v8 release 历史](https://github.com/openai/codex/actions/workflows/rusty-v8-release.yml)、
[release prepare 历史](https://github.com/openai/codex/actions/workflows/rust-release-prepare.yml)、
[stale PR 历史](https://github.com/openai/codex/actions/workflows/close-stale-contributor-prs.yml)。
页面数据会随新运行产生而变化，更新文档时应重新核对。

## 一、整体结构

仓库把 Actions 分成四类：

1. **合并阻断 CI**：`blocking-ci.yml` 是 PR 和 `main` push 的统一入口，
   并行调用 Bazel、Rust、SDK、拼写、依赖许可、仓库规则和 blob 大小检查。
2. **合并后 CI**：`postmerge-ci.yml` 在 `main` push 后运行完整 Rust 测试和
   V8 canary 检查。
3. **发布流水线**：Rust/Codex、zsh、`rusty_v8`、Python runtime、Python SDK
   以及 R2、npm、WinGet、DotSlash 等发布目标。
4. **仓库自动化**：CLA、Issue 翻译、Issue 标签建议、重复 Issue 检测和
   contributor PR 清理。

大量 workflow 仅声明 `workflow_call`，本身是 reusable workflow，必须由
其他 workflow 调用。可直接从 Actions 页面运行的 workflow 主要是带有
`workflow_dispatch` 的 Bazel、完整 Rust CI、V8 canary、release prepare、
Python runtime release 和 stale PR 清理。

### 关键调用关系

```text
blocking-ci (pull_request / main push)
├─ bazel
├─ blob-size-policy
├─ cargo-deny
├─ codespell
├─ repo-checks
├─ rust-ci
└─ sdk
   └─ required 汇总所有子 workflow

postmerge-ci (main push)
├─ rust-ci-full
├─ v8-canary
└─ results 汇总两个子 workflow

rust-release (rust-v* tag)
├─ build → macOS signing/package/finalize
├─ rust-release-windows
├─ rust-release-argument-comment-lint
├─ release
├─ R2 / DotSlash / npm / dev website / WinGet
└─ update-branch

python-sdk-release (python-v* tag)
└─ resolve version → build/publish runtime → build/publish SDK
```

## 二、直接入口 workflow

### `blocking-ci.yml` — 合并阻断 CI

- **用途**：PR 合并前的统一必需检查；`main` push 也运行同一组检查。
- **触发**：`pull_request`；或 push 到 `main`。
- **jobs**：`bazel`、`blob-size-policy`、`cargo-deny`、`codespell`、
  `repo-checks`、`rust-ci`、`sdk`、`required`。
- **流程**：前 7 个 job 并行调用 reusable workflow；`required` 使用
  `.github/scripts/check_ci_results.py` 检查所有依赖结果，并作为 ruleset 要求
  的汇总 job。
- **权限/依赖**：子 workflow 通过 `secrets: inherit` 继承调用者 secrets。
  实际需要的 Secret 由各子 workflow 决定，详见第五节。

### `postmerge-ci.yml` — 合并后完整 CI

- **用途**：`main` 合并后的完整 Rust 测试和 V8 canary 信号，不阻塞 PR 合并。
- **触发**：push 到 `main`。
- **jobs**：`rust-ci-full`、`v8-canary`、`results`。
- **流程**：并行调用完整 Rust CI 与 V8 canary；`results` 使用
  `.github/scripts/check_ci_results.py` 汇总两个结果。
- **说明**：被调用的子 workflow 自己保留 `workflow_dispatch`，维护者可以从
  Actions 页面单独重跑完整套件。

### `bazel.yml` — Bazel CI

- **触发**：`workflow_call`；`workflow_dispatch`。
- **jobs**：
  - `test`：在 macOS、Linux glibc/musl 矩阵中执行 Bazel 全量测试，检查
    `rusty_v8` checksum 和 `MODULE.bazel.lock`，上传日志并保存缓存。
  - `test-windows-shard`：Windows 4 shard 测试。
  - `test-windows`：确认 Windows shards 全部成功。
  - `test-windows-native-main`：仅 `push` 到 `main` 时执行 native Windows
    Bazel 全量测试。
  - `clippy`：Linux/macOS/Windows 多平台 Bazel clippy 检查。
  - `verify-release-build`：验证 release build target，并验证 `bwrap`。
- **触发后的流程**：各平台测试、clippy 和 release-build 验证并行；Windows
  shard 结束后由 `test-windows` 汇总。PR/普通分支使用 shard；`main` 额外获得
  native Windows 信号。
- **Secret**：`BUILDBUDDY_API_KEY`，用于可选的 BuildBuddy/Bazel 远程执行或
  缓存能力。

### `rust-ci-full.yml` — 完整 Rust CI

- **触发**：`workflow_call`；push 到匹配 `**full-ci**` 的开发分支；
  `workflow_dispatch`。
- **jobs**：`general`、`cargo_shear`、`argument_comment_lint_package`、
  `argument_comment_lint_prebuilt`、`lint_build`、
  `tests_macos_aarch64`、`tests_linux_x64_remote`、`tests_linux_arm64`、
  `tests_windows_x64`、`tests_windows_arm64`、`results`。
- **流程**：
  - `general` 做 rustfmt 和 benchmark smoke test；
  - `cargo_shear` 检查未使用依赖；
  - 两个 argument-comment-lint job 分别测试包和预编译 Bazel linter；
  - `lint_build` 在多 target/profile 矩阵执行 clippy/build，使用 Cargo、
    sccache、musl/Zig 和 rusty_v8 artifact；
  - 五个 `tests_*` 调用 `rust-ci-full-nextest-platform.yml`，先生成 nextest
    archive，再跨平台/远程环境分 shard 执行测试；
  - `results` 以 `always()` 汇总所有结果。
- **Secret**：`BUILDBUDDY_API_KEY`。

### `v8-canary.yml` — V8 canary 检查

- **触发**：`workflow_call`、任意 `pull_request`、`workflow_dispatch`。
- **jobs**：`metadata`、`build`、`build-windows-source`。
- **流程**：`metadata` 解析 rusty_v8 版本并运行
  `.github/scripts/v8_canary_changes.py`；只有检测到需要检查时才运行 Linux/
  macOS V8 构建和 Windows source build。无关 PR 只运行便宜的 metadata job。
  产物会做 Cargo smoke test 并上传。
- **Secret**：`BUILDBUDDY_API_KEY`。
- **注意**：workflow 特意不使用 trigger-level path filter，因为同一文件还会
  被 `postmerge-ci` 以 `workflow_call` 调用；变更判断以脚本为准。

### `rust-release.yml` — Codex Rust 发布主流水线

- **触发**：push tag `rust-v*.*.*`。典型操作是创建并推送
  `rust-v0.1.0` 形式的 tag。
- **jobs**：`tag-check`、`build`、`sign-macos-binaries`、`package-macos`、
  `sign-macos-dmg`、`finalize-macos`、`build-windows`、
  `argument-comment-lint-release-assets`、`release`、`publish-r2-assets`、
  `publish-r2`、`publish-dotslash`、`publish-npm`、`deploy-dev-website`、
  `winget`、`update-branch`。
- **流程**：
  1. `tag-check` 确认 tag 版本与 Cargo.toml 一致。
  2. `build` 按 macOS/Linux 多 target 和 primary/app-server bundle 构建，
     生成 symbols、Codex package、Python runtime wheel 和压缩包。
  3. macOS 依次完成 binary 签名/notarization、package、DMG 签名/notarization
     和最终验证。
  4. `build-windows` 调用 Windows reusable workflow，构建、签名、打包 Windows
     二进制、symbols 和 Python wheel。
  5. argument-comment-lint release workflow 仅对正式版本构建发布资产。
  6. `release` 下载所有资产，生成 release notes、checksum、config schema、
     npm 包和 installer，并创建 GitHub Release。
  7. 发布阶段把资产同步到 R2，发布 DotSlash/npm，触发开发网站部署，发布
     WinGet，最后更新 `latest-alpha-cli` 分支。
- **Secret/Variable**：见第五节发布类清单；其中 macOS 签名使用受保护的
  `codesigning` environment，网站部署和 WinGet 也分别需要对应 Secret。

### `rust-release-zsh.yml` — zsh 独立发布

- **触发**：push tag `codex-zsh-v*.*.*`。
- **jobs**：`metadata`、`linux`、`darwin`、`publish-release`。
- **流程**：metadata 校验 tag 且确认 release 不存在；Linux（glibc/musl 的
  x64/arm64）和 Darwin（x64/arm64）构建、smoke test、上传归档；最后创建
  GitHub Release 并发布 DotSlash manifest。
- **固定配置**：`ZSH_COMMIT` 固定上游 zsh commit，`ZSH_PATCH` 指向仓库内的
  `codex-rs/shell-escalation/patches/zsh-exec-wrapper.patch`。
- **Secret**：`GITHUB_TOKEN`（GitHub Release/DotSlash 发布）。

### `rusty-v8-release.yml` — rusty_v8 发布

- **触发**：push tag `rusty-v8-v*.*.*`。
- **jobs**：`metadata`、`build`、`build-windows-source`、`publish-release`。
- **流程**：metadata 解析精确 V8 crate 版本和 release tag；`build` 在 Linux、
  macOS 多架构/沙箱变体中构建 Bazel V8 release pair，并用 Cargo smoke test；
  `build-windows-source` checkout 上游 rusty_v8，构建带 sandbox 的 Windows
  source pair；`publish-release` 下载产物，创建或补充 GitHub Release。
- **Secret**：`BUILDBUDDY_API_KEY`；发布使用 GitHub 内置 token。

### `python-sdk-release.yml` — Python SDK 自动发布

- **触发**：push tag `python-v*`。
- **jobs**：`resolve-python-release`、`prepare-python-runtime`、
  `publish-python-runtime`、`build-python-sdk`、`publish-python-sdk`。
- **流程**：先验证 SDK tag 并解析 runtime/SDK 版本；调用 runtime build 生成
  musllinux wheel；通过 PyPI Trusted Publishing 发布 runtime 并验证可下载；
  构建 SDK package，上传 artifact，再通过 PyPI Trusted Publishing 发布 SDK。
- **权限/Secret**：发布 job 需要 `id-token: write` 以使用 PyPI OIDC；仓库
  内容为 read，不需要 PyPI token Secret。

### `python-runtime-release.yml` — 手动发布 Python runtime

- **触发**：`workflow_dispatch`，必填 input `runtime_version`，例如
  `0.136.0`、`0.136.0a2` 或 `0.136.0a2.post1`。
- **jobs**：`prepare-python-runtime`、`publish-python-runtime`。
- **流程**：调用 runtime build 生成 wheel；下载 artifact，使用 PyPI OIDC
  发布，随后验证 wheel 已在 PyPI 可用。按 runtime version 做并发分组，
  不取消已有运行。
- **权限/Secret**：发布 job 需要 `id-token: write`；仓库内容为 read；不需要
  PyPI token Secret。

### `rust-release-prepare.yml` — 自动更新 release models

- **触发**：`workflow_dispatch`；每 4 小时一次（`0 */4 * * *`）。
- **jobs**：`prepare`。
- **流程**：仅 canonical `openai/codex` 仓库执行；调用 OpenAI API 更新
  `models.json`，若有变化则通过 `peter-evans/create-pull-request` 创建或更新
  PR。
- **Secret/environment**：`CODEX_OPENAI_API_KEY` 映射为 `OPENAI_API_KEY`；使用
  `rust-release-prepare` environment，并需要 contents/pull-requests write。

### `close-stale-contributor-prs.yml` — 清理长期不活跃 contributor PR

- **触发**：`workflow_dispatch`；每天 06:00 UTC（`0 6 * * *`）。
- **jobs**：`close-stale-contributor-prs`。
- **流程**：仅 `openai/codex` 执行，使用 GitHub Script 查找 inactive 14 天的
  contributor PR 并关闭/处理。
- **Secret/权限**：`GITHUB_TOKEN`；contents read、issues write、
  pull-requests write。

### `cla.yml` — CLA Assistant

- **触发**：新建 issue comment；PR `opened`、`closed`、`synchronize`。
- **jobs**：`cla`，仅 repository owner 为 `openai` 时执行。
- **流程**：运行 `contributor-assistant/github-action`，处理贡献者 CLA 状态，
  在需要时评论/更新 PR；合并关闭时保留 CLA 记录。
- **Secret/权限**：`GITHUB_TOKEN`；actions、contents、pull-requests、statuses
  均需 write。

### `issue-translator.yml` — Issue 翻译

- **触发**：Issue `opened`。
- **jobs**：`translate-issue`、`apply-translation`。
- **流程**：只在 `openai/codex` 执行；提取 issue title/body，调用 Codex 生成
  翻译结果；第二个 job 将结果写回 issue。
- **Secret/环境**：`CODEX_OPENAI_API_KEY`；运行于 `issue-triage` environment；
  `translate-issue` 需要 contents read，应用阶段需要 issues write。

### `issue-labeler.yml` — Issue 标签建议

- **触发**：Issue `opened`；或被加上 `codex-label` label。
- **jobs**：`gather-labels`、`apply-labels`。
- **流程**：只在 canonical repo 执行；Codex 根据 issue 内容生成结构化标签
  建议；应用建议后移除触发用的 `codex-label` label，避免循环触发。
- **Secret/环境**：`CODEX_OPENAI_API_KEY`；`issue-triage` environment；使用
  GitHub token 读取 issue 并写入 labels。

### `issue-deduplicator.yml` — 重复 Issue 检测

- **触发**：Issue `opened`；或被加上 `codex-deduplicate` label。
- **jobs**：`gather-duplicates-all`、`normalize-duplicates-all`、
  `gather-duplicates-open`、`normalize-duplicates-open`、`select-final`、
  `comment-on-issue`。
- **流程**：
  1. 第一次 Codex pass 在全部历史 issue 中找候选重复项，并规范化 JSON。
  2. 若没有匹配，再对 open issues 做第二次 pass，并规范化 JSON。
  3. `select-final` 合并两轮输出，最后评论 issue 并移除
     `codex-deduplicate` 触发 label。
- **Secret/环境**：`CODEX_OPENAI_API_KEY`；`issue-triage` environment；需要
  contents read 和最终 issues write。

## 三、Reusable workflow

### `blob-size-policy.yml`

- **触发/输入**：仅 `workflow_call`，无自定义 input。
- **job**：`check`（ubuntu-24.04）。
- **流程**：使用 `openai/fence` audit；完整 checkout；计算比较范围；调用
  `scripts/check_blob_size.py` 检查变更 blob 大小；确认 worktree 干净。
- **Secret/Variable**：无仓库自定义 Secret/Variable；需要 contents read。

### `cargo-deny.yml`

- **触发/输入**：仅 `workflow_call`，无 input。
- **job**：`cargo-deny`。
- **流程**：在 `codex-rs` checkout，运行 `setup-ci`，安装 Rust 1.95.0，执行
  `cargo-deny`，最后检查 worktree。
- **Secret/Variable**：无。

### `codespell.yml`

- **触发/输入**：仅 `workflow_call`，无 input。
- **job**：`codespell`。
- **流程**：checkout，启用 problem matcher，按 `.codespellrc` 执行 codespell，
  检查 worktree。
- **Secret/Variable**：无；workflow 级 contents read。

### `repo-checks.yml`

- **触发/输入**：仅 `workflow_call`，无 input。
- **job**：`build-test`。
- **流程**：验证 Cargo workspace 继承、TUI/core 边界、Bazel clippy flags；运行
  package builder、installer、macOS notarization Python tests；安装 pnpm/Node
  后执行 ASCII、README ToC、Rust format、Prettier 检查；最后检查 worktree。
- **固定变量**：`NODE_OPTIONS=--max-old-space-size=4096`。
- **Secret/Variable**：无。

### `sdk.yml`

- **触发/输入**：仅 `workflow_call`，无 input。
- **jobs**：`python-sdk`、`sdks`。
- **流程**：`python-sdk` 在 glibc Linux image 中测试 Python SDK；`sdks` 安装
  pnpm/Node，使用 Bazel 构建并预热 Codex，安装依赖后构建、lint、测试
  TypeScript SDK。
- **Secret**：`BUILDBUDDY_API_KEY`。

### `python-runtime-build.yml`

- **触发/输入**：仅 `workflow_call`；必填 string input `runtime_version`。
- **job**：`build-python-runtime`，仅 canonical repo 执行。
- **流程**：校验/解析 runtime release，下载已有 release artifacts，构建
  musllinux wheels，上传 `python-runtime-wheels` artifact。
- **Token/权限**：使用 `GITHUB_TOKEN` 下载 GitHub release；contents read。

### `r2-release.yml`

- **触发/输入**：仅 `workflow_call`；必填 inputs：`tag`（string）、
  `make_latest`（boolean）、`prerelease`（boolean）、`stage`（string）。
- **job**：`publish`。
- **流程**：checkout 后执行 `.github/scripts/publish_r2_release.py`（通过环境
  变量接收 tag、latest/pre-release 标志和 stage），把 release assets/metadata
  上传到 Cloudflare R2，并通过 GitHub API 获取 release 信息。
- **Secret/Variable**：`CODEX_R2_ACCESS_KEY_ID`、`CODEX_R2_SECRET_ACCESS_KEY`；
  Variables `CODEX_R2_ENDPOINT_URL`、`CODEX_R2_REGION`；使用 GitHub 内置 token
  读取 release。

### `rust-ci.yml` — PR 快速 Rust CI

- **触发/输入**：`workflow_call`；`workflow_dispatch`。
- **jobs**：`changed`、`general`、`cargo_shear`、
  `argument_comment_lint_package`、`argument_comment_lint_prebuilt`、
  `results`。
- **流程**：`changed` 基于 diff 输出 `codex`、`workflows`、
  `argument_comment_lint`、`argument_comment_lint_package` 四类标志；general
  和 cargo_shear 在 Rust/Codex 代码或 workflow 变化时运行；package lint 按
  package 标志运行；prebuilt lint 以 Linux/macOS/Windows 矩阵执行；results
  汇总所有结果。所有 job 结束后检查 worktree。
- **Secret**：`BUILDBUDDY_API_KEY`（预编译 linter action）。
- **固定环境**：Rust 1.95.0、`CARGO_DYLINT_VERSION=5.0.0`、
  `DYLINT_LINK_VERSION=5.0.0`、`RUST_MIN_STACK=8388608`。

### `rust-ci-full-nextest-platform.yml`

- **触发/输入**：仅 `workflow_call`。必填：`runner`、`target`、`profile`、
  `artifact_id`；可选：`runner_group`、`runner_labels`、`archive_runner`、
  `archive_runner_group`、`archive_runner_labels`、`remote_env`（默认 false）、
  `test_threads`（默认 0）、`use_sccache`（默认 false）。
- **jobs**：`archive`、`shard`、`result`。
- **流程**：archive job 安装工具链/依赖、准备 Cargo/sccache cache，构建并上传
  nextest archive 和 runtime test helpers；shard job 按 1..4 下载产物并行运行
  测试，必要时启动 Docker remote test env；result 在 `always()` 下确认 shards
  成功。
- **Secret**：调用路径可继承 `BUILDBUDDY_API_KEY`；是否使用由 input/runner
  配置决定。

### `rust-release-windows.yml`

- **触发/输入**：仅 `workflow_call`，无 input。
- **jobs**：`build-windows-binaries`、`build-windows-symbols`、`build-windows`。
- **流程**：Windows x64/arm64 构建 primary、helpers、app-server bundles；
  生成 symbols；下载 binaries 后验证并通过 Azure Trusted Signing 签名；构建
  release archives 与 Python runtime wheel 并上传。
- **Secret**：`AZURE_ARTIFACT_SIGNING_CLIENT_ID`、
  `AZURE_ARTIFACT_SIGNING_TENANT_ID`、`AZURE_ARTIFACT_SIGNING_SUBSCRIPTION_ID`、
  `AZURE_ARTIFACT_SIGNING_ENDPOINT`、`AZURE_ARTIFACT_SIGNING_ACCOUNT_NAME`、
  `AZURE_ARTIFACT_SIGNING_CERTIFICATE_PROFILE_NAME`。

### `rust-release-argument-comment-lint.yml`

- **触发/输入**：仅 `workflow_call`；必填 boolean input `publish`。
- **jobs**：`skip`（`publish=false` 时输出跳过信息）、`build`（`publish=true`
  时在 macOS/Linux/Windows 多 target 构建并上传 linter release assets）。
- **固定变量**：`CARGO_NET_GIT_FETCH_WITH_CLI=true`、两个 dylint 版本均为
  `5.0.0`。
- **Secret/Variable**：无。

## 四、各发布/测试流程的输入汇总

| Workflow | 触发方式 | 必填自定义输入 |
| --- | --- | --- |
| `python-runtime-build.yml` | `workflow_call` | `runtime_version: string` |
| `python-runtime-release.yml` | 手动 | `runtime_version: string` |
| `r2-release.yml` | `workflow_call` | `tag`、`stage`（string）；`make_latest`、`prerelease`（boolean） |
| `rust-ci-full-nextest-platform.yml` | `workflow_call` | `runner`、`target`、`profile`、`artifact_id` |
| `rust-release-argument-comment-lint.yml` | `workflow_call` | `publish: boolean` |

其余 workflow 没有用户填写的 workflow input。矩阵中的 target、runner、bundle、
profile、sandbox 等是仓库内固定配置，不是 GitHub Settings 中需要创建的
Variables。

## 五、需要配置的 Secret、Variables 与权限

下面是从 workflow 中实际引用的仓库级 Secret/Variable 汇总。Secret 的具体值
不应写入本文档；还要确保调用 reusable workflow 的 caller 使用了
`secrets: inherit` 或显式传递所需 Secret。

### 必须/按功能配置的 Secret

| Secret | 使用场景 |
| --- | --- |
| `BUILDBUDDY_API_KEY` | Bazel、快速/完整 Rust CI、SDK、V8 canary、rusty_v8 release 的远程执行/缓存 |
| `CODEX_OPENAI_API_KEY` | release prepare、Issue translator、Issue labeler、Issue deduplicator |
| `CODEX_R2_ACCESS_KEY_ID` | R2 发布访问凭据 |
| `CODEX_R2_SECRET_ACCESS_KEY` | R2 发布访问凭据 |
| `AZURE_ARTIFACT_SIGNING_CLIENT_ID` | Windows Azure Trusted Signing |
| `AZURE_ARTIFACT_SIGNING_TENANT_ID` | Windows Azure Trusted Signing |
| `AZURE_ARTIFACT_SIGNING_SUBSCRIPTION_ID` | Windows Azure Trusted Signing |
| `AZURE_ARTIFACT_SIGNING_ENDPOINT` | Windows Azure Trusted Signing |
| `AZURE_ARTIFACT_SIGNING_ACCOUNT_NAME` | Windows Azure Trusted Signing |
| `AZURE_ARTIFACT_SIGNING_CERTIFICATE_PROFILE_NAME` | Windows Azure Trusted Signing |
| `AKV_CODESIGN_RCODESIGN_BLOB_URI` | macOS Azure Key Vault PKCS#11 signing 工具 |
| `AKV_CODESIGN_RCODESIGN_SHA256` | 校验 macOS signing 工具 |
| `AKV_CODESIGN_PKCS11_LIBRARY_BLOB_URI` | macOS Azure Key Vault PKCS#11 library |
| `AKV_CODESIGN_PKCS11_LIBRARY_SHA256` | 校验 PKCS#11 library |
| `AKV_CODESIGN_AZURE_CLIENT_ID` | Azure Key Vault 身份认证 |
| `AKV_CODESIGN_TENANT` | Azure 租户 |
| `AKV_CODESIGN_SUBSCRIPTION` | Azure subscription |
| `AKV_CODESIGN_KEY_VAULT_NAME` | Key Vault 名称 |
| `AKV_CODESIGN_KEY_NAME` | 签名 key 名称 |
| `AKV_CODESIGN_KEY_VERSION` | 可选的签名 key 版本 |
| `AKV_CODESIGN_CERTIFICATE_SHA256` | 可选的签名证书校验值 |
| `AKV_NOTARIZATION_KEY_NAME` | macOS notarization Key Vault key |
| `AKV_NOTARIZATION_KEY_VERSION` | macOS notarization key 版本 |
| `DEV_WEBSITE_VERCEL_DEPLOY_HOOK_URL` | 发布后触发 developers.openai.com 部署 |
| `WINGET_PUBLISH_PAT` | 发布 WinGet package |

### GitHub 自动提供的 token

`GITHUB_TOKEN`/`${{ github.token }}` 不是需要手工创建的普通 Secret，但 workflow
通过 `permissions` 控制它的能力。它用于 checkout、读取/写入 issue、PR、标签、
release、artifact 和分支。若组织规则限制默认 token 权限，需要确认这些
workflow 的显式权限仍被允许；CLA、release、issue 自动化和 branch update 对
write 权限要求最高。

### Repository Variables

| Variable | 用途 |
| --- | --- |
| `CODEX_R2_ENDPOINT_URL` | R2/S3 endpoint，传给 `AWS_ENDPOINT_URL` |
| `CODEX_R2_REGION` | R2/S3 region，传给 `AWS_REGION` |

### GitHub Environments

- `issue-triage`：Issue 翻译、标签和重复检测 job 使用，通常用于隔离
  `CODEX_OPENAI_API_KEY` 并设置审批/保护规则。
- `rust-release-prepare`：models.json 自动更新 job 使用。
- `codesigning`：Rust release 的 macOS binary/DMG 签名与 notarization 使用，
  应配置保护规则和上述 AKV 相关 Secret。

## 六、仓库内辅助 Action 与脚本

workflow 频繁复用以下本地 Action，它们不是独立 workflow，但修改它们会影响
多个 job：

- `setup-ci`：通用 CI 环境和工具准备。
- `setup-bazel-ci` / `prepare-bazel-ci`：Bazel、缓存和 BuildBuddy 相关准备。
- `setup-rusty-v8`：rusty_v8 artifact override 与 checksum 校验。
- `setup-msvc-env`：Windows MSVC/LLVM linker 环境。
- `windows-code-sign` / `linux-code-sign`：Windows Azure 签名和 Linux artifact
  签名。
- `setup-akv-pkcs11-codesigning`：macOS Azure Key Vault PKCS#11 签名工具准备。
- `run-argument-comment-lint`：执行预编译 argument-comment linter。
- `check-clean-worktree`：确保检查或构建没有意外修改仓库。

重要脚本包括：`.github/scripts/check_ci_results.py`、
`.github/scripts/v8_canary_changes.py`、release 打包/签名脚本、
`scripts/check_blob_size.py`、Cargo workspace 校验脚本，以及 Rust/SDK 的
构建脚本。要改变 workflow 的真实行为，不能只看 job 名称，也应同步检查这些
本地 Action 和脚本。

## 七、维护与排障要点

- PR 合并规则应要求 `blocking-ci / required`，而不是分别手工维护每个子 job；
  子 workflow 列表由 `blocking-ci.yml` 中的 `required` job 固定。
- `postmerge-ci` 的失败不会阻塞 PR，但会影响 `main` 的完整测试信号。
- 发布 workflow 都使用 tag 作为版本输入；发布前先确认 Cargo、Python、zsh、
  rusty_v8 的 tag 格式与对应版本文件一致。
- 发布失败最常见的配置问题是 environment 未授权、Azure/R2 Secret 缺失、
  OIDC 的 `id-token: write` 未保留，或 reusable caller 没有继承 Secret。
- issue 自动化在 canonical repository 外会跳过，以避免 fork 使用不到 OpenAI
  API key 或错误修改 issue。
- Rust CI 普遍要求成功后 worktree clean；如果脚本生成了未预期文件，job 会
  在最后一步失败。
