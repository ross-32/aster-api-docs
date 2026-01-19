

## 两套签名方式
### A 管理类接口（main=True）
- 适用：approve/update/delete Agent、approve/update/delete Builder
- 签名：EIP-712（动态 primaryType，如 ApproveAgent/UpdateAgent/DelAgent…）
- 注意：demo 里签名前会把字段名首字母强制大写（例如 `agentName -> AgentName`）。

### B 交易类接口（main=False）
- 适用：下单 `POST /fapi/v3/order`
- 签名：EIP-712 固定 `Message` 类型，仅签 `msg` 字段
- `msg` 内容为“最终 querystring（不含 signature）”
- 关键原则：**签名串 = 最终发送的 querystring（不含 signature）**，任何参数顺序/编码差异都会导致验签失败。

## 共同字段
- `user`：用户主地址（必须）
- `nonce`：防重放（demo 使用微秒格式 nonce）
- 管理类：只需要 `user + nonce + signature + 业务参数`
- 交易类：需要 `user + signer + nonce + signature + 业务参数`

## EIP-712 Domain
- name: `AsterSignTransaction`
- version: `1`
- chainId: `1666`       // mainnet
- chainId: `714`        // testnet
- verifyingContract: `0x0000000000000000000000000000000000000000`

---
