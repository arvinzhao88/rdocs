# Rancher 2.10.6 中 serving-cert 证书过期轮换指南

## Environment
* **Rancher 版本:** v2.10.6
* **集群类型:** 
  * Rancher 创建的下游集群 (Provisioned Clusters)
  * 自导入的下游集群 (Imported Clusters)
  * Local 集群

## Situation
在 Rancher 环境中，系统提示 `serving-cert` 证书即将过期。当前环境中同时存在通过 Rancher 创建的下游集群和自导入的下游集群，以及 Local 集群。需要确认以下问题：
1. 这两种不同类型的下游集群，证书是否会自动轮换？处理方式是否存在差异？
2. Local 集群和下游集群的 `serving-cert` 证书手动轮换的具体操作步骤是什么？

## Resolution
经过在 Rancher 2.10.6 版本中的验证测试，无论是 Rancher 自建的下游集群还是自导入的下游集群，更新/轮换 `serving-cert` 证书的操作流程是一致的。具体步骤如下：

1. **备份当前证书** 
   在进行任何证书操作前，请先通过 `kubectl` 备份当前的 `serving-cert` Secret 及相关集群状态信息，以防需要回滚。

2. **删除现有证书**
   在 Rancher UI 中（或通过 `kubectl delete secret serving-cert -n cattle-system` 命令），定位并删除即将过期的 `serving-cert` 证书资源。

3. **重启 Agent 触发重建**
   执行命令删除 `cattle-cluster-agent` Pod 强制其重启：
   ```bash
   kubectl delete pod -n cattle-system -l app=cattle-cluster-agent
   ```
   
4. **验证证书更新**
   等待 `cattle-cluster-agent` 服务重新启动并恢复为 `Running` 状态。Pod 正常运行后，系统内部机制会自动重新签发并生成新的 `serving-cert` 证书。

## Cause
*注：用户原始输入未包含根本原因，以下为补充的技术分析：*

`serving-cert` 主要是用于 `cattle-cluster-agent` 组件（负责下游集群与 Rancher Server 之间通信的代理）的 TLS 加密通信。默认情况下，这些系统生成的内部证书拥有固定的生命周期（通常为 1 年）。当证书临近过期且未能触发系统的自动轮换机制时，会导致 Agent 与 Server 间通信异常。
通过手动删除过期的 Secret 并重启 Agent Pod，可以利用 Rancher Agent 内置的 Bootstrap/自愈机制：Agent 在启动时若检测到必需的证书 Secret 缺失，会自动向 Rancher Server 发起新的证书签名请求（CSR），从而完成新证书的签发与挂载。