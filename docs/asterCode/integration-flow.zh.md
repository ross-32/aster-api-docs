## **Builder 准备**
   - 准备 `builder` 地址（用于归因与收取 builder fee），这个钱包地址需要有在Aster 注册，并且Aster 合约账户余额里至少有100 Aster。
   - 为每个用户生成一套 API Wallet / Agent（`signer` 地址 + 私钥），私钥由 Builder 后端安全保管（建议每用户独立 signer）。
## **用户授权 Agent（API Wallet）**
   - 调用 `POST /fapi/v3/approveAgent`（用户主钱包签名）
   - 设置权限（现货/合约/提现）、IP 白名单、过期时间等。
## **用户授权 Builder（Builder地址和费率上限）**
   - 调用 `POST /fapi/v3/approveBuilder`（用户主钱包签名）
   - 设置 `maxFeeRate`（builder 可收取的最大费率）。
   **注意：授权 Builder可以在授权Agent时一起授权，也可以在授权Agent后单独授权。**
## **Builder 代用户下单**
   - 用户发起下单
   - Builder 后端调用 `POST /fapi/v3/order`
   - 订单参数中携带 `builder` + `feeRate`
   - 由 `signer` 私钥签名并发送请求。

## **后续维护**

   - 更新 / 撤销 Agent：`updateAgent` / `DELETE /agent`
   - 更新 / 撤销 Builder：`updateBuilder` / `DELETE /builder`

---
