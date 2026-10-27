# Rancher 开启 2FA 后用户重新绑定动态令牌方法

## Environment
* **系统/软件**: Rancher (2.8-ent 以上版本)
* **依赖工具**: Kubernetes 命令行工具 (`kubectl`)
* **权限要求**: 具备 Rancher local (管理) 集群的访问权限及集群管理员权限
* **认证方式**: 双因子认证 (2FA) / 动态令牌

## Situation
Rancher 平台在开启双因子认证（2FA）后，部分用户因手机系统升级、刷机或更换设备，导致绑定的f动态令牌（Authenticator）中的动态验证码丢失。在此场景下，用户无法通过 2FA 验证，进而无法登录 Rancher UI。系统管理员需要针对该特定用户解除现有的 2FA 绑定，使其能够重新登录并配置新的双因子认证令牌。

## Resolution
管理员可以通过 `kubectl` 直接操作 Rancher 管理集群底层的 CRD (Custom Resource Definition) 资源，清理用户的 2FA 配置参数，从而重置其 2FA 状态。具体操作步骤如下：

1. **获取目标用户的 User ID**
   * 使用 `admin` 账号登录 Rancher UI。
   * 进入用户管理（Users）界面，检索无法登录的目标用户。
   * 找到该用户对应的 `User ID`（通常格式如：`u-wzffp`）。

2. **清理用户的 2FA 配置参数**
   * 登录到配置了 `kubectl` 并能够访问 Rancher 管理集群的主机。
   * 执行以下命令，删除该用户命名空间下的 2FA 偏好设置（请将 `<user-id>` 替换为实际的用户 ID）：
   
   ```bash
   kubectl -n <user-id> delete preferences.management.cattle.io enable-two-factor-authenticator
   ```

3. **验证与重新配置 2FA**
   * 上述配置移除后，该用户的 2FA 登录拦截将被解除。
   * 通知用户使用原账号和密码直接登录 Rancher UI。
   * 登录成功后，系统会再次弹出 2FA 配置引导，用户可使用新的动态令牌扫描二维码完成重新绑定。

## Cause
双因子认证的 TOTP (Time-based One-Time Password) 密钥硬绑定在用户的本地移动设备中。当设备环境发生变化（如升级、更换）导致密钥丢失时，用户端将无法生成与服务端匹配的动态验证码。

在 Rancher 的底层架构设计中，用户的 2FA 开启状态及其相关配置，是以自定义资源（CRD）`preferences.management.cattle.io` 的形式，存储在以该用户 ID 命名的 Namespace 下的。通过 `kubectl` 删除该命名空间下的 `enable-two-factor-authenticator` 记录，可以强制清空服务端的 2FA 激活状态，从而触发 Rancher 的安全机制，引导用户在下次登录时进入初始的重新绑定流程。