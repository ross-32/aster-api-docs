## 接口通用说明

* 所有接口均采用 JSON-RPC 2.0 协议，通过 HTTP POST 方式调用。
* 接口基础地址：**https://tapi.asterdex.com/info**
* 所有请求需设置请求头 `Content-Type: application/json`。
* `blockTag` 参数传入 `"latest"` 表示查询最新状态。
* 所有时间相关字段均为毫秒级时间戳。

> **注意：** 在 Open Orders 和 Fills 接口中，隐私模式仅影响开启后的新订单；非隐私模式下完成的历史订单仍按正常规则展示。

---
