# 参与贡献

感谢你愿意改进 HealthWatch。这个项目同时涉及移动端、BLE 和 ESP32，提交前请尽量说明测试环境。

## 开始之前

- 先搜索现有 Issue，避免重复提交。
- Bug 请注明手机型号、Android/iOS 版本、ESP32 型号和固件 BLE 配置。
- 涉及真实健康数据或电话号码时，请先脱敏。
- 安全漏洞和隐私问题不要附带真实个人数据公开提交。

## 本地开发

1. Fork 并克隆仓库。
2. 使用 HBuilderX 打开项目。
3. 准备支持 BLE 的真机和 ESP32 测试设备。
4. 确认 `pages/home/home.vue` 中的 UUID 与固件一致。
5. 使用 Android App 基座进行真机调试。

## 提交规范

建议使用简洁的提交前缀：

- `feat:` 新功能
- `fix:` Bug 修复
- `docs:` 文档更新
- `refactor:` 不改变功能的重构
- `test:` 测试相关
- `chore:` 工程配置

示例：`fix: avoid duplicate BLE listeners after reconnect`

## Pull Request 检查

- 说明改动目的和用户可见影响；
- 标注已测试的平台、设备和 ESP32 固件配置；
- UI 改动附上截图或录屏；
- BLE 协议改动同步更新 `docs/DEVICE_PROTOCOL.md`；
- 新增权限时解释用途和必要性；
- 确认未提交电话号码、健康数据、密钥或构建产物；
- 保持改动聚焦，避免混入无关格式化。

## 代码方向

欢迎贡献：

- BLE 稳定性、重连和错误提示；
- 健康数据持久化与导出；
- 无障碍与界面体验；
- 数据协议兼容性；
- 自动化测试与文档；
- Android 新版本权限适配。
