# 开发与操作记录

本文档用于记录 `owoow` 项目的开发事项、Git 操作和远程仓库变更。

## 记录规范

新增记录时，建议包含以下信息：

- 日期
- 类型（开发、修复、配置或仓库操作）
- 分支
- 操作内容
- 执行结果
- 必要的后续事项

## 操作记录

### 2026-09-12：配置 `private` 远程并发布 `develop`

- 类型：仓库配置与分支发布
- 分支：`develop`
- 远程名称：`private`
- 远程地址：`git@github1.com:969981/owoow.git`
- 执行内容：
  1. 将远程地址配置为 `private`。
  2. 确认远程已有 `master`，尚未存在 `develop`。
  3. 执行 `git push -u private develop`。
- 结果：推送成功，远程自动创建 `develop` 分支，本地 `develop` 已关联跟踪 `private/develop`。
- 备注：当前环境通过 Windows OpenSSH 使用 `github1.com` 的 SSH 配置；如 Linux shell 未自动读取该配置，可使用：

  ```bash
  GIT_SSH_COMMAND=/mnt/c/Windows/System32/OpenSSH/ssh.exe git push private develop
  ```

## 开发记录

后续开发事项可按以下格式追加：

### YYYY-MM-DD：事项标题

- 类型：开发 / 修复 / 重构
- 分支：`branch-name`
- 变更：
  - 描述变更内容。
- 验证：
  - 描述测试或验证方式。
- 结果：描述完成情况及后续事项。
