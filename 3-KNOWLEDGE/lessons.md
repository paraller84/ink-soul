# 技术教训

## Python 导入
- 连字符文件名 → 不能用标准 import，用 importlib.util.spec_from_file_location()

## 状态迁移
- 必须幂等：重复执行不会产生副作用
- 必须兼容旧路径：迁移后旧路径不能挂

## Flask Blueprint
- DispatchingJinjaLoader 按注册顺序搜索模板目录
- 同名模板文件冲突 → Blueprint 模板目录必须唯一
- Dual-registration Blueprint（同时注册到独立 app 和 edu-hub）→ 硬编码路径失效，必须用 url_for()

## 大上下文自动分叉
- 工具调用 > 20 且信息获取 < 50% 时自动委托子代理
- 检测到 patch×2 + terminal×3 调试循环 → delegate_task 隔离试错

## 记忆管理
- memory 仅存跨会话事实（偏好/环境/约定/架构决策/路径/有价值观察）
- 容量目标 ≤ 5K/30 条
- 版本日志/Bug 细节走 session_search，技术陷阱走 skill
- 每月 1 日自动瘦身
