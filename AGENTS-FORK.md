# RootMyS24 Fork 维护 Agent 指南

本文件记录该仓库 fork 后的长期维护方式，供后续接手的人或 AI 使用。
根目录 `AGENTS.md` 仍然有效；本文件只补充 fork、CI、分支和发布约定。

## 当前目标

- 这是个人自用 fork，GitHub 仓库为 `pentalogos/root-my-s24`。
- 自用 APK 优先从 GitHub Actions 或本机构建，签名必须与已安装自用版一致，才能覆盖升级。
- 上游功能修复和新固件适配要能持续跟进，不因为个人改动而无法同步。
- 功能改动保留成可提上游 PR 的干净分支；**未经用户明确要求，不创建、不更新 PR**。

## 远程与分支模型

```text
upstream  https://github.com/NanoTurtle1145/root-my-s24.git
origin    https://github.com/pentalogos/root-my-s24.git

upstream/main        原仓库，只读同步源
origin/main          个人 fork 主分支 = 自用集成版
feat/ksu-disable     可选的上游 PR 分支，只包含可提交的功能改动
```

规则：

- `origin/main` 是自用构建分支，可以包含 endpoint 置空、内置 KSU 兜底等个人改动。
- `feat/ksu-disable` 从 `upstream/main` 拉出，只放通用功能，不混入自用 endpoint、签名、文档偏好。
- 不再需要长期保留 `selfuse/combined`；它的内容应合入 `origin/main` 后删除。
- 更新上游时，把 `upstream/main` 合入或 rebase 到 `main`，不要反过来把个人改动推给上游。

## 提交划分

通用功能提交（可进 `feat/*` PR）：

- `RootViewModel.kt`：late-load 前禁用 KSU 模块、helper `-c` 调用修复。
- `SettingsScreen.kt`：设置开关。
- `values*/strings.xml`：对应文案。

自用提交（只留在 `main`）：

- `RootViewModel.kt`：KSU 3.3.0 内置兜底（DZF2 / CZB2_CHC / S9210DZI1）。
- `FirmwareSelectActivity.kt`：KSU 版本不可用提示。
- `LogUploader.kt`：`DEFAULT_ENDPOINT = ""`。
- `PayloadStatus.kt`：`ENDPOINT = ""`。

内置 KSU 兜底和对应 UI 提示只服务非官方构建，**不要放进 `feat/*` PR**，否则会让上游 PR 混入
自用构建需求。不要把以上两类混成一个准备提 PR 的提交。自用提交可以留在 `main`，不要放进
`feat/*`。

## 同步上游

```bash
git fetch upstream
git switch main
git merge upstream/main
# 或：git rebase upstream/main
git push origin main
```

如果上游以后合并了同类功能：

- 先解决冲突；
- 确认 `main` 不再保留重复实现；
- `feat/*` 分支用 `git rebase upstream/main` 更新，不要 rebase 到个人 `main`。

## GitHub Actions 自编译

当前仓库没有上游 workflow，需要自己维护 `.github/workflows/selfuse-build.yml`。

要求：

- 使用 `actions/checkout` 且 `fetch-depth: 0`。
- 构建前执行 `git fetch origin main:refs/remotes/origin/main`，因为 `versionCode` 来自该 ref；否则可能退回 `1`。
- 执行 `git fetch --tags --force`，否则 `versionName` 取不到 tag。
- JDK 21；Android SDK 需要 `platforms;android-37.0`、`build-tools;37.0.0`、`ndk;29.0.14206865`、`cmake;3.22.1`。
- 使用 `android-actions/setup-android@v4`；v3 会尝试安装已下线的 `tools` 包并失败。
- 已验证 workflow 能完成构建；未配置签名 secrets 时 `Configure signing` 会跳过，产物是 debug 签名。
- 当前 fork 已配置签名 secrets，使用项目专属 `rms24` 自用证书；workflow 会用 `apksigner verify --print-certs` 输出证书 SHA-256。该证书 SHA-256 为 `13:7F:B4:63:1C:76:FF:60:6F:40:44:5B:F0:B4:F4:12:28:A7:CC:03:9A:1C:D8:E6:8A:51:F3:6E:89:42:63:2F`。
- 签名通过 GitHub Secrets 注入，绝不提交 keystore 或密码：
  - `ANDROID_KEYSTORE_BASE64`
  - `ANDROID_STORE_PASSWORD`
  - `ANDROID_KEY_ALIAS`
  - `ANDROID_KEY_PASSWORD`
- 没有签名 secret 时 Gradle 会回退 debug 签名；这种 APK 不能覆盖已安装的自用 release 版，只适合临时测试。
- APK 作为 Actions artifact 或 GitHub Release 附件，不提交进 git。
- 当前触发器：`push` 到 `main`（忽略 Markdown 和 `docs/`）+ 手动 `workflow_dispatch`。
  跟进上游后推 `main` 即自动构建，不再依赖本地编译。

`versionCode` 注意：

- 构建脚本执行 `git rev-list --count refs/remotes/origin/main`。
- 因为 `origin/main` 是自用集成分支，个人提交会增加 versionCode。
- 如果 `origin` 仍指向上游、个人改动只在别的分支，versionCode 不会反映这些提交。
- 发 APK 前先 `git fetch origin`，确认本地 `origin/main` 是最新 fork 主分支。

## 构建产物处置

绝不提交：

- `build/`、`app/build/`、`manager-ui/build/`、`services/*/build/`
- `.gradle/`、`.kotlin/`
- `local.properties`
- `*.keystore`、`*.jks`、`*.idsig`
- `~/.gradle/gradle.properties`
- Downloads 里的 APK、AAB、日志

仓库中已有的 `app/src/main/assets/bakasu-manager.apk`、`libadb.so`、payload 文件是源码资产，不是本次编译产物，保留。

### 本地只保留仓库

- 不在本地执行 `assembleRelease`；所有构建走 GitHub Actions。
- 本地只保留 git 仓库、源码和文档，`build/`、`.gradle/`、`.kotlin/`、`local.properties`、`.DS_Store` 一律清掉。
- 跟进上游后只需 `git fetch upstream && git merge upstream/main && git push origin main`，
  Actions 自动构建，从 run 页面下载 artifact。
- 不把 APK 拉回本地长期保存；需要安装时下载后直接 `adb install -r`，随后删除。
- 清理命令：

```bash
rm -rf build app/build manager-ui/build services/*/build .gradle .kotlin
rm -f local.properties .DS_Store app/.DS_Store manager-ui/.DS_Store services/.DS_Store
```

需要分发 APK 时：

```bash
APK="$HOME/Downloads/RootMyS24-....apk"
shasum -a 256 "$APK" > "$APK.sha256"
gh release create <tag> "$APK" "$APK.sha256" \
  --repo pentalogos/root-my-s24 \
  --target main \
  --title "..." --notes "..."
```

**未经用户明确要求，不创建 tag、不发布 Release。**

## 签名与包名

- 自用 keystore 只在本地和 GitHub Secrets 中保存，不进入仓库。
- 公开 fork 的源码可以公开，但签名私钥不能公开。
- 当前 `applicationId` 仍是 `cn.nanoturtle.rootmys9280`，与官方包同 ID 但签名不同；用户不能直接覆盖安装官方版，需要先卸载。
- 如果以后要作为独立应用公开分发，再讨论是否改 `applicationId` 和应用名；不要擅自改。

## 许可证

仓库是 GPL-3.0。公开 fork、发布 APK 或再分发时必须保留 `LICENSE`，并保证对应源码可获取。

## Agent 硬性规则

1. 不提交构建产物、keystore、密码、`local.properties`、`.DS_Store`。
2. 不擅自创建或更新 PR；用户明确要求后才操作。
3. 不擅自改 `applicationId`、`BuildOrigin.OFFICIAL_CERT_SHA256`、官方 endpoint 或包名。
4. 不把自用 endpoint 改动混入 `feat/*`。
5. 不改写 `origin` 的历史；只允许对个人 feature 分支使用 `--force-with-lease`。
6. 发布 Release/tag 前必须得到用户确认。
7. 每次改完代码至少构建一次 `:app:assembleRelease`，并检查 `git status --short` 只包含预期文件。
