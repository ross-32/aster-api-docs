## **Two signature modes**

### A Management endpoints (`main=True`)
- Applies to: approve/update/delete Agent; approve/update/delete Builder
- Signature: EIP-712 (dynamic `primaryType`, e.g., ApproveAgent / UpdateAgent / DelAgent …)
- Note: In the demo, field names are uppercased for the first letter before signing (e.g., `agentName -> AgentName`).

### B Trading endpoints (`main=False`)
- Applies to: placing orders `POST /fapi/v3/order`
- Signature: EIP-712 fixed `Message` type, signing only the `msg` field
- `msg` is the “final querystring (excluding `signature`)”
- Key rule: **the signed string must equal the final sent querystring (excluding signature)**. Any difference in parameter ordering/encoding will cause signature verification to fail.

## **Common fields**
- `user`: user main address (required)
- `nonce`: anti-replay nonce (demo uses microseconds-style nonce)
- Management endpoints: require `user + nonce + signature + business params`
- Trading endpoints: require `user + signer + nonce + signature + business params`

### EIP-712 Domain
- name: `AsterSignTransaction`
- version: `1`
- chainId: `1666`       // mainnet
- chainId: `714`        // testnet
- verifyingContract: `0x0000000000000000000000000000000000000000`

---
