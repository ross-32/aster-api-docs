## Builder preparation
   - Prepare the builder address (used for attribution and collecting the builder fee). This wallet address must be registered on Aster, and the Aster contract account must have a balance of at least 100 ASTER.
   - Generate an API Wallet / Agent for each user (`signer` address + private key). The private key must be securely stored on the Builder backend (recommended: one signer per user).

## User authorizes the Agent (API Wallet)
   - Call `POST /fapi/v3/approveAgent` (signed by the user’s main wallet)
   - Configure permissions (spot/perp/withdraw), IP whitelist, expiry, etc.

## User authorizes the Builder (builder address and fee cap)
   - Call `POST /fapi/v3/approveBuilder` (signed by the user’s main wallet)
   - Set `maxFeeRate` (maximum fee rate the builder is allowed to charge).
   - **Note: Builder authorization can be done together during Agent approval, or separately after Agent approval.**

## Builder places orders on behalf of the user
   - The user initiates the order action (e.g., clicks “Place Order” in the Builder UI)
   - Builder backend calls `POST /fapi/v3/order`
   - Include `builder` + `feeRate` in the order parameters
   - Sign and send the request using the `signer` private key.

## Ongoing maintenance
   - Update / revoke Agent: `updateAgent` / `DELETE /agent`
   - Update / revoke Builder: `updateBuilder` / `DELETE /builder`

---
