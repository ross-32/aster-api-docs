## General API Information

* All endpoints use JSON-RPC 2.0 protocol via HTTP POST.
* The base endpoint is: **https://tapi.asterdex.com/info**
* All requests must set `Content-Type: application/json`.
* The `blockTag` parameter accepts `"latest"` to query the most recent state.
* All time-related fields are in milliseconds.

> **Note:** In the Open Orders and Fills endpoints, account privacy enabled only applies to new orders placed after it is enabled; orders completed when account privacy is disabled remain visible as usual.

---
