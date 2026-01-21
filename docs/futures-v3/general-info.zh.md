## **Rest 基本信息**

* 接口可能需要用户的AGENT
* AGENT创建请[点击这里](https://www.asterdex.com/zh-CN/api-wallet) 并且切换到API管理上方的`专业API`
* 本篇列出REST接口的baseurl **https://fapi.asterdex.com**
* 所有接口的响应都是JSON格式
* 响应中如有数组，数组元素以时间升序排列，越早的数据越提前。
* 所有时间、时间戳均为UNIX时间，单位为毫秒
* 所有数据类型采用JAVA的数据类型定义

### HTTP 返回代码
* HTTP `4XX` 错误码用于指示错误的请求内容、行为、格式。
* HTTP `403` 错误码表示违反WAF限制(Web应用程序防火墙)。
* HTTP `429` 错误码表示警告访问频次超限，即将被封IP
* HTTP `418` 表示收到429后继续访问，于是被封了。
* HTTP `5XX` 错误码用于指示Aster Finance服务侧的问题。    
* HTTP `503` 表示API服务端已经向业务核心提交了请求但未能获取响应，特别需要注意的是其不代表请求失败，而是未知。很可能已经得到了执行，也有可能执行失败，需要做进一步确认。


### 接口错误代码
* 每个接口都有可能抛出异常

> 异常响应格式如下：

```javascript
{
  "code": -1121,
  "msg": "Invalid symbol."
}
```

* 具体的错误码及其解释在**错误码**页面

### 接口的基本信息
* `GET`方法的接口, 参数必须在`query string`中发送.
* `POST`, `PUT`, 和 `DELETE` 方法的接口, 在 `request body`中发送(content type `application/x-www-form-urlencoded`)
* 对参数的顺序不做要求。

## **访问限制**
* 在 `/fapi/v3/exchangeInfo`接口中`rateLimits`数组里包含有REST接口(不限于本篇的REST接口)的访问限制。包括带权重的访问频次限制、下单速率限制。本篇`枚举定义`章节有限制类型的进一步说明。
* 违反上述任何一个访问限制都会收到HTTP 429，这是一个警告.

<aside class="notice">
请注意，若用户被认定利用频繁挂撤单且故意低效交易意图发起攻击行为，Aster Finance有权视具体情况进一步加强对其访问限制。
</aside>


### IP 访问限制
* 每个请求将包含一个`X-MBX-USED-WEIGHT-(intervalNum)(intervalLetter)`的头，其中包含当前IP所有请求的已使用权重。
* 每个路由都有一个"权重"，该权重确定每个接口计数的请求数。较重的接口和对多个交易对进行操作的接口将具有较重的"权重"。
* 收到429时，您有责任作为API退回而不向其发送更多的请求。
* **如果屡次违反速率限制和/或在收到429后未能退回，将导致API的IP被禁(http状态418)。**
* 频繁违反限制，封禁时间会逐渐延长 ，**对于重复违反者，将会被封从2分钟到3天**。
* **访问限制是基于IP的，而不是AGENT**

<aside class="notice">
强烈建议您尽可能多地使用websocket消息获取相应数据,既可以保障消息的及时性，也可以减少请求带来的访问限制压力。
</aside>


### 下单频率限制
* 每个下单请求回报将包含一个`X-MBX-ORDER-COUNT-(intervalNum)(intervalLetter)`的头，其中包含当前账户已用的下单限制数量。
* 被拒绝或不成功的下单并不保证回报中包含以上头内容。
* **下单频率限制是基于每个账户计数的。**

**关于交易时效性** 
互联网状况并不100%可靠，不可完全依赖,因此你的程序本地到服务器的时延会有抖动.
这是我们设置`recvWindow`的目的所在，如果你从事高频交易，对交易时效性有较高的要求，可以灵活设置recvWindow以达到你的要求。

<aside class="notice">
不推荐使用5秒以上的recvWindow
</aside>

## **接口鉴权类型**
* 每个接口都有自己的鉴权类型，鉴权类型决定了访问时应当进行何种鉴权
* 如果需要鉴权，应当在请求体中添加signer

鉴权类型 | 描述
------------ | ------------
NONE | 不需要鉴权的接口
TRADE | 需要有效的signer和签名
USER_DATA | 需要有效的signer和签名
USER_STREAM | 需要有效的signer和签名
MARKET_DATA | 需要有效的signer和签名

## **鉴权签名体**
参数 | 描述
------------ | ------------
user | 主账户钱包地址
signer | API钱包地址
nonce | 当前时间戳,单位为微秒
signature | 签名

## **需要签名的接口**
* TRADE 与 USER_DATA,USER_STREAM,MARKET_DATA
* 接口参数转字符串后按照key值ASCII编码后生成的字符串 请注意所有参数取值请以字符串的方式进行签名
* 生成字符串后在与鉴权签名参数的user,signer,nonce使用web3的abi参数编码生成字节码
* 生成字节码后使用Keccak算法生成hash
* 使用派生地址的私钥用web3的ecdsa签名算法对该hash进行签名生成signature

### 时间同步安全
* 签名接口均需要传递`timestamp`参数,其值应当是请求发送时刻的unix时间戳(毫秒)
* 服务器收到请求时会判断请求中的时间戳,如果是5000毫秒之前发出的,则请求会被认为无效。这个时间窗口值可以通过发送可选参数`recvWindow`来自定义。

> 逻辑伪代码：
  
  ```javascript
  if (timestamp < (serverTime + 1000) && (serverTime - timestamp) <= recvWindow) {
    // process request
  } else {
    // reject request
  }
  ```

## **POST /fapi/v3/order 的示例**

#### 所有参数均通过from body请求(Python 3.9.6)

#### 示例 : 以下参数为api注册信息,user,signer,privateKey仅供示范(privateKey为signer的私钥)


Key | Value | Desc
------------ | ------------ | ------------
user | 0x63DD5aCC6b1aa0f563956C0e534DD30B6dcF7C4e  | 登陆钱包地址
signer | 0x21cF8Ae13Bb72632562c6Fff438652Ba1a151bb0 | [点击这里获取](https://www.asterdex.com/zh-CN/api-wallet)
privateKey | 0x4fd0a42218f3eae43a6ce26d22544e986139a01e5b34a62db53757ffca81bae1 | [点击这里获取](https://www.asterdex.com/zh-CN/api-wallet)

#### 示例 : nonce参数为当前系统微秒值,超过系统时间,或者落后系统时间超过5s为非法请求
```python
#python
nonce = math.trunc(time.time()*1000000)
print(nonce)
#1748310859508867
```
```java
//java
Instant now = Instant.now();
long microsecond = now.getEpochSecond() * 1000000 + now.getNano() / 1000;
```

#### 示例 : 以下参数为业务请求参数 

```python
    my_dict = {'symbol': 'SANDUSDT', 'positionSide': 'BOTH', 'type': 'LIMIT', 'side': 'BUY',
	         'timeInForce': 'GTC', 'quantity': "190", 'price': 0.28694}
```

#### 示例 : 所有参数通过 form body 发送(方法以python为例) 

> **第一步将所有业务参数转字符串后按照ascII排序生成字符串:**

```python
    #定义所有元素取值转换为字符串
    def _trim_dict(my_dict) :
    # 假设待删除的字典为d
     for key in my_dict:
        value = my_dict[key]
        if isinstance(value, list):
            new_value = []
            for item in value:
                if isinstance(item, dict):
                    new_value.append(json.dumps(_trim_dict(item)))
                else:
                    new_value.append(str(item))
            my_dict[key] = json.dumps(new_value)
            continue
        if isinstance(value, dict):
            my_dict[key] = json.dumps(_trim_dict(value))
            continue
        my_dict[key] = str(value)

    return my_dict

    #移除空值元素
    my_dict = {key: value for key, value in my_dict.items() if  value is not None}
    my_dict['recvWindow'] = 50000
    my_dict['timestamp'] = int(round(time.time()*1000))
    # my_dict['timestamp'] = 1749545309665
    #将元素转换为字符串
    _trim_dict(my_dict)
    #根据ASCII排序生成字符串并移除特殊字符
    json_str = json.dumps(my_dict, sort_keys=True).replace(' ', '').replace('\'','\"')
    print(json_str)
    {"positionSide":"BOTH","price":"0.28694","quantity":"190","recvWindow":"50000","side":"BUY","symbol":"SANDUSDT","timeInForce":"GTC","timestamp":"1749545309665","type":"LIMIT"}
```

> **第二步将第一步生成的字符串与账户信息以及nonce进行abi编码生成hash字符串:**

```python
   from eth_abi import encode
   from web3 import Web3
   #使用WEB3 ABI对生成的字符串和user, signer, nonce进行编码
   encoded = encode(['string', 'address', 'address', 'uint256'], [json_str, user, signer, nonce])
   print(encoded.hex())
   #000000000000000000000000000000000000000000000000000000000000008000000000000000000000000063dd5acc6b1aa0f563956c0e534dd30b6dcf7c4e00000000000000000000000021cf8ae13bb72632562c6fff438652ba1a151bb00000000000000000000000000000000000000000000000000006361457bcec8300000000000000000000000000000000000000000000000000000000000000af7b22706f736974696f6e53696465223a22424f5448222c227072696365223a22302e3238363934222c227175616e74697479223a22313930222c227265637657696e646f77223a223530303030222c2273696465223a22425559222c2273796d626f6c223a2253414e4455534454222c2274696d65496e466f726365223a22475443222c2274696d657374616d70223a2231373439353435333039363635222c2274797065223a224c494d4954227d0000000000000000000000000000000000
   #keccak hex
   keccak_hex =Web3.keccak(encoded).hex()
   print(keccak_hex)
   #9e0273fc91323f5cdbcb00c358be3dee2854afb2d3e4c68497364a2f27a377fc
```
> **第三步将第二步生成的hash用privateKey进行签名:**
```python
    from eth_account import Account
    from eth_abi import encode
    from web3 import Web3, EthereumTesterProvider
    from eth_account.messages import encode_defunct

    signable_msg = encode_defunct(hexstr=keccak_hex)
    signed_message = Account.sign_message(signable_message=signable_msg, private_key=priKey)
    signature =  '0x'+signed_message.signature.hex()
    print(signature)
    #0x0337dd720a21543b80ff861cd3c26646b75b3a6a4b5d45805d4c1d6ad6fc33e65f0722778dd97525466560c69fbddbe6874eb4ed6f5fa7e576e486d9b5da67f31b
```
> **第四步将所有参数以及第三步生成的signature组装成请求体:**

```python
    my_dict['nonce'] = nonce
    my_dict['user'] = user
    my_dict['signer'] = signer
    my_dict['signature'] = '0x'+signed_message.signature.hex()
    url ='https://fapi.asterdex.com/fapi/v3/order'
    headers = {
            'Content-Type': 'application/x-www-form-urlencoded',
            'User-Agent': 'PythonApp/1.0'
    }
    res = requests.post(url,data=my_dict,headers=headers)
    print(url)
    #curl  -X POST 'https://fapi.asterdex.com/fapi/v3/order' -d 'symbol=SANDUSDT&positionSide=BOTH&type=LIMIT&side=BUY&timeInForce=GTC&quantity=190&price=0.28694&recvWindow=50000&timestamp=1749545309665&nonce=1748310859508867&user=0x63DD5aCC6b1aa0f563956C0e534DD30B6dcF7C4e&signer=0x21cF8Ae13Bb72632562c6Fff438652Ba1a151bb0&signature=0x0337dd720a21543b80ff861cd3c26646b75b3a6a4b5d45805d4c1d6ad6fc33e65f0722778dd97525466560c69fbddbe6874eb4ed6f5fa7e576e486d9b5da67f31b'   

```

## **GET /fapi/v3/order 的示例**
#### 示例 : 所有参数通过 query string 发送(Python 3.9.6) 

#### 示例 : 以下参数为api注册信息,user,signer,privateKey仅供示范(privateKey为agent的私钥)

Key | Value
------------ | ------------
user | 0x63DD5aCC6b1aa0f563956C0e534DD30B6dcF7C4e
signer | 0x21cF8Ae13Bb72632562c6Fff438652Ba1a151bb0
privateKey | 0x4fd0a42218f3eae43a6ce26d22544e986139a01e5b34a62db53757ffca81bae1

#### 示例 : nonce参数为当前系统微秒值,超过系统时间50s,或者落后系统时间超过5s为非法请求
```python
#python
nonce = math.trunc(time.time()*1000000)
print(nonce)
#1748310859508867
```
```java
//java
Instant now = Instant.now();
long microsecond = now.getEpochSecond() * 1000000 + now.getNano() / 1000;
```

#### 示例 : 以下参数为业务请求参数 
```python
my_dict = {'symbol':'SANDUSDT','side':"SELL","type":'LIMIT','orderId':2194215}
```

> **第一步将所有业务参数转字符串后按照ascII排序生成字符串:**

```python
    #定义所有元素取值转换为字符串
    def _trim_dict(my_dict) :
      for key in my_dict:
        value = my_dict[key]
        if isinstance(value, list):
            new_value = []
            for item in value:
                if isinstance(item, dict):
                    new_value.append(json.dumps(_trim_dict(item)))
                else:
                    new_value.append(str(item))
            my_dict[key] = json.dumps(new_value)
            continue
        if isinstance(value, dict):
            my_dict[key] = json.dumps(_trim_dict(value))
            continue
        my_dict[key] = str(value)

    return my_dict

    #移除空值元素
    my_dict = {key: value for key, value in my_dict.items() if  value is not None}
    my_dict['recvWindow'] = 50000
    my_dict['timestamp'] = int(round(time.time()*1000))
    # my_dict['timestamp'] = 1749545309665
    #将元素转换为字符串
    _trim_dict(my_dict)
    #根据ASCII排序生成字符串并移除特殊字符
    json_str = json.dumps(my_dict, sort_keys=True).replace(' ', '').replace('\'','\"')
    print(json_str)
    #{"orderId":"2194215","recvWindow":"50000","side":"BUY","symbol":"SANDUSDT","timestamp":"1749545309665","type":"LIMIT"}
```

> **第二步将第一步生成的字符串与账户信息以及nonce进行abi编码生成hash字符串:**

```python
   from eth_abi import encode
   from web3 import Web3

   #使用WEB3 ABI对生成的字符串和user, signer, nonce进行编码
   encoded = encode(['string', 'address', 'address', 'uint256'], [json_str, user, signer, nonce])
   print(encoded.hex())
   #000000000000000000000000000000000000000000000000000000000000008000000000000000000000000063dd5acc6b1aa0f563956c0e534dd30b6dcf7c4e00000000000000000000000021cf8ae13bb72632562c6fff438652ba1a151bb00000000000000000000000000000000000000000000000000006361457bcec8300000000000000000000000000000000000000000000000000000000000000767b226f726465724964223a2232313934323135222c227265637657696e646f77223a223530303030222c2273696465223a22425559222c2273796d626f6c223a2253414e4455534454222c2274696d657374616d70223a2231373439353435333039363635222c2274797065223a224c494d4954227d00000000000000000000
   keccak_hex =Web3.keccak(encoded).hex()
   print(keccak_hex)
   #6ad9569ea1355bf62de1b09b33b267a9404239af6d9227fa59e3633edae19e2a
```
> **第三步将第二步生成的hash用privateKey进行签名:**
```python
    from eth_account import Account
    from eth_abi import encode
    from web3 import Web3, EthereumTesterProvider
    from eth_account.messages import encode_defunct

    signable_msg = encode_defunct(hexstr=keccak_hex)
    signed_message = Account.sign_message(signable_message=signable_msg, private_key=priKey)
    signature =  '0x'+signed_message.signature.hex()
    print(signature)
    #0x4f5e36e91f0d4cf5b29b6559ebc2c808d3c808ebb13b2bcaaa478b98fb4195642c7473f0d1aa101359aaf278126af1a53bcb482fb05003bfb6bdc03de03c63151b
```
> **第四步将所有参数以及第三步生成的signature组装成请求体:**

```python
    my_dict['nonce'] = nonce
    my_dict['user'] = user
    my_dict['signer'] = signer
    my_dict['signature'] = '0x'+signed_message.signature.hex()

    url ='https://fapi.asterdex.com/fapi/v3/order'

    res = requests.get(url, params=my_dict)
    print(url)
    #curl  -X GET 'https://fapi.asterdex.com/fapi/v3/order?symbol=SANDUSDT&side=BUY&type=LIMIT&orderId=2194215&recvWindow=50000&timestamp=1749545309665&nonce=1748310859508867&user=0x63DD5aCC6b1aa0f563956C0e534DD30B6dcF7C4e&signer=0x21cF8Ae13Bb72632562c6Fff438652Ba1a151bb0&signature=0x4f5e36e91f0d4cf5b29b6559ebc2c808d3c808ebb13b2bcaaa478b98fb4195642c7473f0d1aa101359aaf278126af1a53bcb482fb05003bfb6bdc03de03c63151b'

```
## **完整python脚本示例**
```python
#Python 3.9.6
#eth-account~=0.13.7
#eth-abi~=5.2.0
#web3~=7.11.0
#requests~=2.32.3

import json
import math
import time
import requests

from eth_abi import encode
from eth_account import Account
from eth_account.messages import encode_defunct
from web3 import Web3

user = '0x63DD5aCC6b1aa0f563956C0e534DD30B6dcF7C4e'
signer='0x21cF8Ae13Bb72632562c6Fff438652Ba1a151bb0'
priKey = "0x4fd0a42218f3eae43a6ce26d22544e986139a01e5b34a62db53757ffca81bae1"
host = 'https://fapi.asterdex.com'
placeOrder = {'url': '/fapi/v3/order', 'method': 'POST',
              'params':{'symbol': 'SANDUSDT', 'positionSide': 'BOTH', 'type': 'LIMIT', 'side': 'BUY',
	         'timeInForce': 'GTC', 'quantity': "30", 'price': 0.325,'reduceOnly': True}}
getOrder = {'url':'/fapi/v3/order','method':'GET','params':{'symbol':'SANDUSDT','side':"BUY","type":'LIMIT','orderId':2194215}}

def call(api):
    nonce = math.trunc(time.time() * 1000000)
    my_dict = api['params']
    send(api['url'],api['method'],sign(my_dict,nonce))

def sign(my_dict,nonce):
    my_dict = {key: value for key, value in my_dict.items() if  value is not None}
    my_dict['recvWindow'] = 50000
    my_dict['timestamp'] = int(round(time.time()*1000))
    msg = trim_param(my_dict,nonce)
    signable_msg = encode_defunct(hexstr=msg)
    signed_message = Account.sign_message(signable_message=signable_msg, private_key=priKey)
    my_dict['nonce'] = nonce
    my_dict['user'] = user
    my_dict['signer'] = signer
    my_dict['signature'] = '0x'+signed_message.signature.hex()

    print(my_dict['signature'])
    return  my_dict

def trim_param(my_dict,nonce) -> str:
    _trim_dict(my_dict)
    json_str = json.dumps(my_dict, sort_keys=True).replace(' ', '').replace('\'','\"')
    print(json_str)
    encoded = encode(['string', 'address', 'address', 'uint256'], [json_str, user, signer, nonce])
    print(encoded.hex())
    keccak_hex =Web3.keccak(encoded).hex()
    print(keccak_hex)
    return keccak_hex

def _trim_dict(my_dict) :
    for key in my_dict:
        value = my_dict[key]
        if isinstance(value, list):
            new_value = []
            for item in value:
                if isinstance(item, dict):
                    new_value.append(json.dumps(_trim_dict(item)))
                else:
                    new_value.append(str(item))
            my_dict[key] = json.dumps(new_value)
            continue
        if isinstance(value, dict):
            my_dict[key] = json.dumps(_trim_dict(value))
            continue
        my_dict[key] = str(value)

    return my_dict

def send(url, method, my_dict):
    url = host + url
    print(url)
    print(my_dict)
    if method == 'POST':
        headers = {
            'Content-Type': 'application/x-www-form-urlencoded',
            'User-Agent': 'PythonApp/1.0'
        }
        res = requests.post(url, data=my_dict, headers=headers)
        print(res.text)
    if method == 'GET':
        res = requests.get(url, params=my_dict)
        print(res.text)
    if method == 'DELETE':
        res = requests.delete(url, data=my_dict)
        print(res.text)

if __name__ == '__main__':
    call(placeOrder)
    # call(getOrder)

```

## **公开API参数**
### 术语解释
* `base asset` 指一个交易对的交易对象，即写在靠前部分的资产名
* `quote asset` 指一个交易对的定价资产，即写在靠后部分资产名


### 枚举定义

**交易对类型:**

* FUTURE 期货

**合约类型 (contractType):**

* PERPETUAL 永续合约


**合约状态 (contractStatus, status):**

* PENDING_TRADING   待上市
* TRADING          	交易中
* PRE_SETTLE			预结算
* SETTLING			结算中
* CLOSE				已下架


**订单状态 (status):**

* NEW 新建订单
* PARTIALLY_FILLED  部分成交
* FILLED  全部成交
* CANCELED  已撤销
* REJECTED 订单被拒绝
* EXPIRED 订单过期(根据timeInForce参数规则)

**订单种类 (orderTypes, type):**

* LIMIT 限价单
* MARKET 市价单
* STOP 止损限价单
* STOP_MARKET 止损市价单
* TAKE_PROFIT 止盈限价单
* TAKE_PROFIT_MARKET 止盈市价单
* TRAILING_STOP_MARKET 跟踪止损单

**订单方向 (side):**

* BUY 买入
* SELL 卖出

**持仓方向:**

* BOTH 单一持仓方向
* LONG 多头(双向持仓下)
* SHORT 空头(双向持仓下)

**有效方式 (timeInForce):**

* GTC - Good Till Cancel 成交为止
* IOC - Immediate or Cancel 无法立即成交(吃单)的部分就撤销
* FOK - Fill or Kill 无法全部立即成交就撤销
* GTX - Good Till Crossing 无法成为挂单方就撤销

**条件价格触发类型 (workingType)**

* MARK_PRICE
* CONTRACT_PRICE 

**响应类型 (newOrderRespType)**

* ACK
* RESULT

**K线间隔:**

m -> 分钟; h -> 小时; d -> 天; w -> 周; M -> 月

* 1m
* 3m
* 5m
* 15m
* 30m
* 1h
* 2h
* 4h
* 6h
* 8h
* 12h
* 1d
* 3d
* 1w
* 1M

**限制种类 (rateLimitType)**

> REQUEST_WEIGHT

```javascript
  {
  	"rateLimitType": "REQUEST_WEIGHT",
  	"interval": "MINUTE",
  	"intervalNum": 1,
  	"limit": 2400
  }
```

> ORDERS

```javascript
  {
  	"rateLimitType": "ORDERS",
  	"interval": "MINUTE",
  	"intervalNum": 1,
  	"limit": 1200
   }
```

* REQUESTS_WEIGHT  单位时间请求权重之和上限

* ORDERS    单位时间下单(撤单)次数上限


**限制间隔**

* MINUTE



## **过滤器**
过滤器，即Filter，定义了一系列交易规则。
共有两类，分别是针对交易对的过滤器`symbol filters`，和针对整个交易所的过滤器`exchange filters`(暂不支持)

### 交易对过滤器
#### PRICE_FILTER 价格过滤器

> **/exchangeInfo 响应中的格式:**

```javascript
  {
    "filterType": "PRICE_FILTER",
    "minPrice": "0.00000100",
    "maxPrice": "100000.00000000",
    "tickSize": "0.00000100"
  }
```

价格过滤器用于检测order订单中price参数的合法性

* `minPrice` 定义了 `price`/`stopPrice` 允许的最小值
* `maxPrice` 定义了 `price`/`stopPrice` 允许的最大值。
* `tickSize` 定义了 `price`/`stopPrice` 的步进间隔，即price必须等于minPrice+(tickSize的整数倍)
以上每一项均可为0，为0时代表这一项不再做限制。

逻辑伪代码如下：

* `price` >= `minPrice`
* `price` <= `maxPrice`
* (`price`-`minPrice`) % `tickSize` == 0



#### LOT_SIZE 订单尺寸

> */exchangeInfo 响应中的格式:**

```javascript
  {
    "filterType": "LOT_SIZE",
    "minQty": "0.00100000",
    "maxQty": "100000.00000000",
    "stepSize": "0.00100000"
  }
```

lots是拍卖术语，这个过滤器对订单中的`quantity`也就是数量参数进行合法性检查。包含三个部分：

* `minQty` 表示 `quantity` 允许的最小值.
* `maxQty` 表示 `quantity` 允许的最大值
* `stepSize` 表示 `quantity`允许的步进值。

逻辑伪代码如下：

* `quantity` >= `minQty`
* `quantity` <= `maxQty`
* (`quantity`-`minQty`) % `stepSize` == 0


#### MARKET_LOT_SIZE 市价订单尺寸
参考LOT_SIZE，区别仅在于对市价单还是限价单生效

#### MAX_NUM_ORDERS 最多订单数


> **/exchangeInfo 响应中的格式:**

```javascript
  {
    "filterType": "MAX_NUM_ORDERS",
    "limit": 200
  }
```

定义了某个交易对最多允许的挂单数量(不包括已关闭的订单)

普通订单与条件订单均计算在内


#### MAX_NUM_ALGO_ORDERS 最多条件订单数

> **/exchangeInfo format:**

```javascript
  {
    "filterType": "MAX_NUM_ALGO_ORDERS",
    "limit": 100
  }
```

定义了某个交易对最多允许的条件订单的挂单数量(不包括已关闭的订单)。   

条件订单目前包括`STOP`, `STOP_MARKET`, `TAKE_PROFIT`, `TAKE_PROFIT_MARKET`, 和 `TRAILING_STOP_MARKET`


#### PERCENT_PRICE 价格振幅过滤器

> **/exchangeInfo 响应中的格式:**

```javascript
  {
    "filterType": "PERCENT_PRICE",
    "multiplierUp": "1.1500",
    "multiplierDown": "0.8500",
    "multiplierDecimal": 4
  }
```

`PERCENT_PRICE` 定义了基于标记价格计算的挂单价格的可接受区间.

挂单价格必须同时满足以下条件：

* 买单: `price` <= `markPrice` * `multiplierUp`
* 卖单: `price` >= `markPrice` * `multiplierDown`


#### MIN_NOTIONAL 最小名义价值

> **/exchangeInfo 响应中的格式:**

```javascript
  {
    "filterType": "MIN_NOTIONAL",
    "notioanl": "1"
  }
```

MIN_NOTIONAL过滤器定义了交易对订单所允许的最小名义价值(成交额)。
订单的名义价值是`价格`*`数量`。 
由于`MARKET`订单没有价格，因此会使用 mark price 计算。   






---

