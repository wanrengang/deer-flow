# UniClaw 维护指南

本文档记录了如何将 DeerFlow 定制为 UniClaw，以及如何后续同步官方更新。

---

## 一、当前定制改动清单

共 33 个文件被修改：

### Backend (1 个文件)
| 文件 | 改动说明 |
|------|----------|
| `backend/Dockerfile` | Node.js 安装源改为 npmmirror.com（解决国内无法访问 nodesource.com） |

### Frontend - 组件 (13 个文件)
| 文件 | 改动说明 |
|------|----------|
| `frontend/src/app/layout.tsx` | 应用标题 "DeerFlow" → "UniClaw" |
| `frontend/src/components/landing/header.tsx` | 导航栏名称 |
| `frontend/src/components/landing/footer.tsx` | 页脚名称 |
| `frontend/src/components/landing/hero.tsx` | Hero 区域名称 |
| `frontend/src/components/landing/progressive-skills-animation.tsx` | Agent 名称、"Ask DeerFlow" → "Ask UniClaw" |
| `frontend/src/components/landing/sections/case-study-section.tsx` | "See how DeerFlow" → "See how UniClaw" |
| `frontend/src/components/landing/sections/community-section.tsx` | 社区板块名称 |
| `frontend/src/components/landing/sections/sandbox-section.tsx` | Sandbox 板块名称 |
| `frontend/src/components/landing/sections/skills-section.tsx` | Skills 板块名称 |
| `frontend/src/components/landing/sections/whats-new-section.tsx` | What's New 板块名称 |
| `frontend/src/components/workspace/settings/about-content.ts` | About 页面内容 |
| `frontend/src/components/workspace/settings/about.md` | About 页面标题 |
| `frontend/src/components/workspace/workspace-header.tsx` | Workspace 头部名称 |

### Frontend - 内容 (11 个文件)
| 文件 | 改动说明 |
|------|----------|
| `frontend/src/content/en/index.mdx` | 文档首页 |
| `frontend/src/content/en/introduction/*.mdx` | Introduction 文档（why-deerflow, core-concepts, harness-vs-app）|
| `frontend/src/content/en/harness/index.mdx` | Harness 文档 |
| `frontend/src/content/en/application/index.mdx` | Application 文档 |
| `frontend/src/content/en/tutorials/deploy-your-own-deerflow.mdx` | 教程文档 |
| `frontend/src/content/en/posts/weekly/2026-04-06.mdx` | 周报 |
| `frontend/src/content/zh/index.mdx` | 中文首页 |
| `frontend/src/content/zh/posts/releases/2_0_rc.mdx` | 中文发布说明 |
| `frontend/src/content/zh/posts/weekly/2026-04-06.mdx` | 中文周报 |

### Frontend - 元数据 (6 个文件)
| 文件 | 改动说明 |
|------|----------|
| `frontend/src/content/en/_meta.ts` | "DeerFlow Harness" → "UniClaw Harness", "DeerFlow App" → "UniClaw App" |
| `frontend/src/content/en/introduction/_meta.ts` | "Why DeerFlow" → "Why UniClaw" |
| `frontend/src/content/en/tutorials/_meta.ts` | "Deploy Your Own DeerFlow" → "Deploy Your Own UniClaw" |
| `frontend/src/core/agents/api.ts` | 错误信息 "Could not reach the DeerFlow backend" → "Could not reach the UniClaw backend" |
| `frontend/src/core/i18n/locales/en-US.ts` | 英文 i18n 文本 |
| `frontend/src/core/i18n/locales/zh-CN.ts` | 中文 i18n 文本 |

---

## 二、导出定制改动

### 方法 1：导出完整 Patch

```bash
# 导出所有改动为 patch 文件
git diff HEAD > ~/uniclaw-customizations.patch

# 查看改动统计
git diff --stat HEAD
```

### 方法 2：创建定制分支

```bash
# 创建 uniclaw 分支，基于当前所有改动
git checkout -b uniclaw
git push -u origin uniclaw
```

---

## 三、同步官方更新的工作流

### Step 1: 添加上游仓库（如果还没有）

```bash
# 检查当前 remote
git remote -v

# 添加上游仓库（如果 origin 只指向官方仓库）
git remote add upstream https://github.com/bytedance/deer-flow.git

# 如果 origin 指向你自己的 fork
git remote add upstream https://github.com/bytedance/deer-flow.git
git remote set-url origin https://github.com/YOUR_USERNAME/deer-flow.git
```

### Step 2: 同步官方最新代码

```bash
# 确保在 main 分支
git checkout main

# 拉取官方最新代码
git fetch upstream
git merge upstream/main

# 如果有冲突，解决冲突后继续
# git add .
# git commit -m "Merge upstream/main"
```

### Step 3: 应用 UniClaw 定制

**方式 A：重新应用 patch**
```bash
# 先暂存本地改动
git stash

# 合并官方代码
git merge upstream/main

# 应用定制 patch
git apply ~/uniclaw-customizations.patch

# 手动检查冲突
git diff --name-only --diff-filter=U
```

**方式 B：通过 merge 定制分支**
```bash
# 假设你已将改动保存到 uniclaw 分支
git checkout main
git merge upstream/main   # 合并官方
git merge uniclaw        # 合并定制（可能需要解决冲突）
```

### Step 4: 验证改动

```bash
# 检查文件状态
git status

# 运行构建测试
cd frontend && pnpm build
cd ../backend && make lint && make test
```

---

## 四、增量更新的建议

由于官方 DeerFlow 的核心代码（backend）可能频繁更新，而你的定制主要是品牌相关的文本替换，建议：

1. **保留原始的 backend 目录**，只修改 `Dockerfile`（Node.js 镜像源）
2. **记录每个改动的文件**，方便选择性同步
3. **使用 `git diff HEAD -- filename`** 对比单个文件的改动

---

## 五、快速参考命令

```bash
# 查看所有定制文件
git diff --name-only HEAD

# 查看改动统计
git diff --stat HEAD

# 导出 patch
git diff HEAD > ~/uniclaw-customizations.patch

# 应用 patch
git apply ~/uniclaw-customizations.patch

# 查看官方仓库最新 commit
git log upstream/main --oneline -5

# 检查是否能干净地 merge
git merge-base --is-ancestor upstream/main HEAD && echo "OK: main is up to date" || echo "WARN: main is behind"
```

---

## 六、注意事项

1. **backend/Dockerfile** 的改动（Node.js 镜像源）可能需要随官方 Dockerfile 结构调整
2. **frontend/src/content/** 下的文档内容可能与官方文档结构不同步
3. 建议定期同步，避免积累过多冲突

---

## 七、当前仓库配置

### Remote 配置
```
origin  → https://github.com/wanrengang/deer-flow.git  (你的 fork)
upstream → https://github.com/bytedance/deer-flow.git   (官方仓库)
```

### 最新提交
```
commit 8b2a5180
feat: rename DeerFlow to UniClaw - branding customization
  32 files changed, 379 insertions(+), 187 deletions(-)
```

### Fork 地址
- 你的 Fork：https://github.com/wanrengang/deer-flow
- 官方仓库：https://github.com/bytedance/deer-flow

---

## 八、日常维护工作流

### 场景 1：官方有新版本，你想同步代码

```bash
# 1. 确保在 main 分支，且没有未提交的改动
git checkout main
git status

# 2. 拉取官方最新代码并合并
git fetch upstream
git merge upstream/main

# 3. 如果有冲突，解决冲突后：
git add .
git commit -m "merge: resolve conflicts with upstream"

# 4. 推送到你的 fork
git push origin main
```

### 场景 2：你想继续开发新功能

```bash
# 1. 基于 main 创建新分支
git checkout -b feature/your-feature-name

# 2. 开发完成后，提交到你的 fork
git add .
git commit -m "feat: description"
git push origin feature/your-feature-name

# 3. 在 GitHub 上创建 PR 合并到你的 main
```

### 场景 3：查看改动

```bash
# 查看所有改动
git diff HEAD

# 查看某个文件改动
git diff HEAD -- frontend/src/app/layout.tsx

# 查看提交历史
git log --oneline -10
```

---

## 九、相关链接

- [UniClaw Fork 仓库](https://github.com/wanrengang/deer-flow)
- [DeerFlow 官方仓库](https://github.com/bytedance/deer-flow)
- [维护指南文档](docs/UNICLAW_MAINTENANCE_GUIDE.md)

---

## 七、获取帮助

如果同步过程中遇到问题：
1. 查看 git 冲突标记 `<<<<<<<`, `=======`, `>>>>>>>`
2. 使用 `git diff` 对比改动
3. 参考 [DeerFlow 官方文档](https://deerflow.tech)
