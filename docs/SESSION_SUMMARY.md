# UniClaw 项目会话总结

**最后更新**: 2026-04-19
**项目路径**: D:\study\deer-flow

---

## 项目概述

将 DeerFlow (字节跳动的开源 Agent 框架) 定制为 UniClaw 品牌。

**当前状态**: 品牌定制已完成，推送到 GitHub fork

---

## Git 配置

| 配置项 | 值 |
|--------|-----|
| origin | https://github.com/wanrengang/deer-flow.git |
| upstream | https://github.com/bytedance/deer-flow.git |
| 当前分支 | main |
| 最新 commit | fb9febd0 |

---

## 已完成工作

### 1. Docker Build 修复
- 修改 `backend/Dockerfile`
- 将 Node.js 安装源从 nodesource.com 改为 npmmirror.com（解决国内访问问题）

### 2. UniClaw 品牌定制 (33 个文件)
- Frontend 组件名称替换 (DeerFlow → UniClaw)
- i18n 文本替换
- 文档内容替换
- 错误消息替换

### 3. Git 仓库配置
- Fork 官方仓库到 wanrengang/deer-flow
- 添加 upstream remote
- 提交所有改动

---

## Docker 环境状态

```
容器名                    状态     端口
deer-flow-frontend       运行中   3000
deer-flow-gateway        运行中   8001
deer-flow-langgraph      运行中   2024
deer-flow-nginx          运行中   2026
```

**访问地址**: http://localhost:2026

---

## 后续工作

待完成（用户后续会安排）：
1. 官方 DeerFlow 更新后的代码同步
2. UniClaw 品牌继续完善
3. 功能定制开发

---

## 快速命令

```bash
# 查看改动
git status

# 查看维护指南
cat docs/UNICLAW_MAINTENANCE_GUIDE.md

# 同步官方更新
git fetch upstream
git merge upstream/main
git push origin main
```

---

## 联系人/账号

- GitHub 用户: wanrengang
- Fork 地址: https://github.com/wanrengang/deer-flow
