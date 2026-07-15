# GitLab 自动跟随 GitHub —— 配置指南

目标：让 GitLab 仓库（`gitlab.roboticplus.com:robimweld/awesomeweldoneskills`）
成为 GitHub 仓库（`github.com:Naoki326/awesomeweldoneskills`）的镜像，
由 GitLab CI 流水线（`.gitlab-ci.yml` 中的 `mirror-from-github` 任务）定时同步。

GitHub 是唯一事实来源（source of truth），GitLab 只读跟随。

---

## 工作原理

```
   GitHub (origin, 权威)          GitLab (镜像, 跟随)
        │                              ▲
        │  git clone --mirror          │
        └──────────► CI 流水线 ────────┘
                  (schedule)   git push --mirror
```

定时（或手动）触发的流水线：
1. 用 GitHub deploy key 以 SSH 方式 `clone --mirror` 拉取全部分支/标签；
2. 以 `push --mirror` 推回本 GitLab 仓库，使两者引用完全一致。

普通 push 不会触发本任务（仅 `schedule` 与 `web`），因此镜像回推不会自我循环。

---

## 前置条件

- GitLab Runner 所在环境能访问公网 `github.com`（出方向）。
  （若 Runner 处于内网且无法出公网，则改用 GitHub Actions 方案，见文末。）

---

## 一次性配置步骤

### 1. 生成 GitHub 只读 deploy key

```bash
ssh-keygen -t ed25519 -C "gitlab-mirror" -f gitlab_mirror_key -N ""
```

把 **公钥** `gitlab_mirror_key.pub` 加入 GitHub 仓库：
`github.com/Naoki326/awesomeweldoneskills` → **Settings → Deploy keys → Add**
→ 勾选只读（**不要**勾 Allow write access）。私钥妥善保管。

### 2. 把私钥写入 GitLab CI 变量

`gitlab.roboticplus.com/robimweld/awesomeweldoneskills` →
**Settings → CI/CD → Variables → Add variable**：

| Key | Value | 选项 |
|-----|-------|------|
| `GITHUB_DEPLOY_KEY` | 私钥全文（含 `-----BEGIN...-----` / `END...`） | Masked（不可加 Protected，除非目标分支受保护且 Runner 受保护） |

> 若私钥含 `+` 等被 Masked 拦截的字符导致报错，改用 **文件型变量** `GITHUB_DEPLOY_KEY_FILE`（Type = File），流水线会自动识别。

### 3. （推荐）创建可删除分支的 GitLab Token

`--mirror` 会删除 GitLab 上 GitHub 已不存在的分支，需要写+删权限，
`CI_JOB_TOKEN` 通常无法删除分支，因此推荐：

**Settings → Repository → Access Tokens → Add**：
- Name: `mirror-bot`
- Role: Reporter 或 Developer
- Scopes: `write_repository`
- 复制生成的 token，加入 CI 变量 `GITLAB_TOKEN`（Masked）。

> 留空 `GITLAB_TOKEN` 时流水线回退到 `CI_JOB_TOKEN`，此时若删除分支被拒，
> 把 `.gitlab-ci.yml` 里 `git push --mirror` 改成
> `git push --force --all gitlab && git push --force --tags gitlab`（不删除 GitLab 独有分支）。

### 4. 引导：让 `.gitlab-ci.yml` 出现在 GitLab 目标分支

调度只会在 **含 `.gitlab-ci.yml` 的分支** 上运行。首次需手动把 GitHub 的
`main` 推到 GitLab（在本机执行，需对 GitLab 有写权限）：

```bash
git fetch origin main
git push gitlab origin/main:main
```

随后在 GitLab **Settings → Repository → Default branch** 把默认分支改为 `main`。

### 5. 创建定时调度

**Build → Pipeline schedules → New schedule**：
- Description: `mirror-from-github`
- Interval Pattern: 例如 `*/30 * * * *`（每 30 分钟）
- Target branch: `main`
- 勾选 **Activated**

保存后可点 **Run pipeline** 立即验证一次。

---

## 验证

- 在 **Build → Pipelines** 看到类型为 `schedule` 的成功任务；
- GitLab 侧出现 `main` 分支，且内容/提交与 GitHub 一致；
- 后续 GitHub 推送后，下一个调度周期 GitLab 自动跟上。

---

## 分支说明

本仓库历史上 GitLab 默认分支为 `master`，GitHub 默认分支为 `main`，
两边 `master` 曾各自演进。GitHub `main` 已是二者合集（同一逻辑改动以不同
SHA 各自存在），因此全量镜像覆盖 GitLab 不会丢失实际内容。镜像后 GitLab
默认分支统一为 `main`。

---

## 备选方案（Runner 无法出公网时）

改用 GitHub Actions 在 GitHub 侧（有公网）触发、推送到 GitLab（需 GitLab 对
GitHub Actions 可达）：

```yaml
# .github/workflows/mirror-to-gitlab.yml
name: mirror-to-gitlab
on: [push, delete]
jobs:
  mirror:
    runs-on: ubuntu-latest
    steps:
      - run: |
          git clone --mirror https://github.com/${{ github.repository }}.git repo
          cd repo
          git push --mirror "https://oauth2:${{ secrets.GITLAB_TOKEN }}@gitlab.roboticplus.com/robimweld/awesomeweldoneskills.git"
```

> GitLab token 存于 GitHub Secrets，此时同步是实时（push 即触发）的，但凭据落在 GitHub 侧。

另：GitLab 自带 **Settings → Repository → Mirror** 的 pull mirror 功能（非流水线），
最省事，可按需替代本方案。
