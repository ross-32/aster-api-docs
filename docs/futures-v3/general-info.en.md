## Overview

* This API may require the user's AGENT. 
* To create an AGENT please [click here](https://www.asterdex.com/en/api-wallet) and switch to `Pro API` at the top
* The base endpoint is: **https://fapi.asterdex.com**
* All endpoints return either a JSON object or array.
* Data is returned in **ascending** order. Oldest first, newest last.
* All time and timestamp related fields are in milliseconds.
* All data types adopt definition in JAVA.

### HTTP Return Codes

* HTTP `4XX` return codes are used for for malformed requests;
  the issue is on the sender's side.
* HTTP `403` return code is used when the WAF Limit (Web Application Firewall) has been violated.
* HTTP `429` return code is used when breaking a request rate limit.
* HTTP `418` return code is used when an IP has been auto-banned for continuing to send requests after receiving `429` codes.
* HTTP `5XX` return codes are used for internal errors; the issue is on
  Aster's side.
* HTTP `503` return code is used when the API successfully sent the message but not get a response within the timeout period.
  It is important to **NOT** treat this as a failure operation; the execution status is
  **UNKNOWN** and could have been a success.

### Error Codes and Messages

* Any endpoint can return an ERROR

> ***The error payload is as follows:***

```javascript
{
  "code": -1121,
  "msg": "Invalid symbol."
}
```

* Specific error codes and messages defined in [Error Codes](#error-codes).

### General Information on Endpoints

* For `GET` endpoints, parameters must be sent as a `query string`.
* For POST, PUT, and DELETE method APIs, send data in the request body (content type application/x-www-form-urlencoded)
* Parameters may be sent in any order.

## LIMITS

* The `/fapi/v3/exchangeInfo` `rateLimits` array contains objects related to the exchange's `RAW_REQUEST`, `REQUEST_WEIGHT`, and `ORDER` rate limits. These are further defined in the `ENUM definitions` section under `Rate limiters (rateLimitType)`.
* A `429` will be returned when either rate limit is violated.

<aside class="notice">
Aster Finance has the right to further tighten the rate limits on users with intent to attack.
</aside>

### IP Limits

* Every request will contain `X-MBX-USED-WEIGHT-(intervalNum)(intervalLetter)` in the response headers which has the current used weight for the IP for all request rate limiters defined.
* Each route has a `weight` which determines for the number of requests each endpoint counts for. Heavier endpoints and endpoints that do operations on multiple symbols will have a heavier `weight`.
* When a 429 is received, it's your obligation as an API to back off and not spam the API.
* **Repeatedly violating rate limits and/or failing to back off after receiving 429s will result in an automated IP ban (HTTP status 418).**
* IP bans are tracked and **scale in duration** for repeat offenders, **from 2 minutes to 3 days**.
* **The limits on the API are based on the IPs, not the API keys.**

<aside class="notice">
It is strongly recommended to use websocket stream for getting data as much as possible, which can not only ensure the timeliness of the message, but also reduce the access restriction pressure caused by the request.
</aside>

### Order Rate Limits

* Every order response will contain a `X-MBX-ORDER-COUNT-(intervalNum)(intervalLetter)` header which has the current order count for the account for all order rate limiters defined.
* Rejected/unsuccessful orders are not guaranteed to have `X-MBX-ORDER-COUNT-**` headers in the response.
* **The order rate limit is counted against each account**.

**Serious trading is about timing.** Networks can be unstable and unreliable,
which can lead to requests taking varying amounts of time to reach the
servers. With `recvWindow`, you can specify that the request must be
processed within a certain number of milliseconds or be rejected by the
server.

<aside class="notice">
It is recommended to use a small recvWindow of 5000 or less!
</aside>

## API authentication type

* Each API has its own authentication type, which determines what kind of authentication is required when accessing it.
* If authentication is required, a signer should be included in the request body.

| Security Type | Description                               |
| ------------- | ----------------------------------------- |
| NONE          | API that does not require authentication  |
| TRADE         | A valid signer and signature are required |
| USER_DATA     | A valid signer and signature are required |
| USER_STREAM   | A valid signer and signature are required |
| MARKET_DATA   | A valid signer and signature are required |

## Authentication signature payload

| Parameter | Description                        |
| --------- | ---------------------------------- |
| user      | Main account wallet address        |
| signer    | API wallet address                 |
| nonce     | Current timestamp, in microseconds |
| signature | Signature                          |

## Endpoints requiring signature 
* Security Type: TRADE, USER_DATA, USER_STREAM, MARKET_DATA
* After converting the API parameters to strings, sort them by their key values in ASCII order to generate the final string. Note: All parameter values must be treated as strings during the signing process.
* After generating the string, combine it with the authentication signature parameters user, signer, and nonce, then use Web3’s ABI parameter encoding to generate the bytecode.
* After generating the bytecode, use the Keccak algorithm to generate the hash.
* Use the private key of **API wallet address** to sign the hash using web3’s ECDSA signature algorithm, generating the final signature.

### Timing Security

* A `SIGNED` endpoint also requires a parameter, `timestamp`, to be sent which
  should be the millisecond timestamp of when the request was created and sent.
* An additional parameter, `recvWindow`, may be sent to specify the number of
  milliseconds after `timestamp` the request is valid for. If `recvWindow`
  is not sent, **it defaults to 5000**.

> The logic is as follows:

```javascript
if (timestamp < (serverTime + 1000) && (serverTime - timestamp) <= recvWindow){
    // process request
  } 
  else {
    // reject request
  }
```

## Example of POST /fapi/v3/order

#### All parameters are passed through the request body (Python 3.9.6)

#### The following parameters are API registration details. The values for user, signer, and privateKey are for demonstration purposes only (the privateKey corresponds to the signer).

| Key        | Value          | Desc                                                    
| ---------- | ------------------------------------------------------------------ |  ---------- |
| user       | 0x63DD5aCC6b1aa0f563956C0e534DD30B6dcF7C4e                         |Login wallet address |
| signer     | 0x21cF8Ae13Bb72632562c6Fff438652Ba1a151bb0                         |[Click Here](https://www.asterdex.com/en/api-wallet)         | 
| privateKey | 0x4fd0a42218f3eae43a6ce26d22544e986139a01e5b34a62db53757ffca81bae1 |[Click Here](https://www.asterdex.com/en/api-wallet)        | 

#### The nonce parameter is the current system time in microseconds. If it exceeds the system time or lags behind it by more than 5 seconds, the request is considered invalid.

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
#### Example: The following parameters are business request parameters.

```python
my_dict = {'symbol': 'SANDUSDT', 'positionSide': 'BOTH', 'type': 'LIMIT', 'side': 'BUY',
	         'timeInForce': 'GTC', 'quantity': "190", 'price': 0.28694}
```

#### Example: All parameters are sent via form body (using Python as an example).

> **Step 1: Convert all business parameters to strings, then sort them in ASCII order to generate a string.**

```python
    #Define all element values as strings.
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

    #Remove elements with null (empty) values.
    my_dict = {key: value for key, value in my_dict.items() if  value is not None}
    my_dict['recvWindow'] = 50000
    my_dict['timestamp'] = int(round(time.time()*1000))
    # my_dict['timestamp'] = 1749545309665
    #Convert elements to strings.
    _trim_dict(my_dict)
    #Generate a string sorted by ASCII values and remove special characters.
    json_str = json.dumps(my_dict, sort_keys=True).replace(' ', '').replace('\'','\"')
    print(json_str)
    #{"positionSide":"BOTH","price":"0.28694","quantity":"190","recvWindow":"50000","side":"BUY","symbol":"SANDUSDT","timeInForce":"GTC","timestamp":"1749545309665","type":"LIMIT"}
```

> **Step 2: Take the string generated in Step 1 and encode it together with the account information and nonce using ABI encoding to generate a hash string.**

```python
   from eth_abi import encode
   from web3 import Web3
   #Use Web3 ABI to encode the generated string along with user, signer, and nonce.
   encoded = encode(['string', 'address', 'address', 'uint256'], [json_str, user, signer, nonce])
   print(encoded.hex())
   #000000000000000000000000000000000000000000000000000000000000008000000000000000000000000063dd5acc6b1aa0f563956c0e534dd30b6dcf7c4e00000000000000000000000021cf8ae13bb72632562c6fff438652ba1a151bb00000000000000000000000000000000000000000000000000006361457bcec8300000000000000000000000000000000000000000000000000000000000000af7b22706f736974696f6e53696465223a22424f5448222c227072696365223a22302e3238363934222c227175616e74697479223a22313930222c227265637657696e646f77223a223530303030222c2273696465223a22425559222c2273796d626f6c223a2253414e4455534454222c2274696d65496e466f726365223a22475443222c2274696d657374616d70223a2231373439353435333039363635222c2274797065223a224c494d4954227d0000000000000000000000000000000000
   #keccak hex
   keccak_hex =Web3.keccak(encoded).hex()
   print(keccak_hex)
   #9e0273fc91323f5cdbcb00c358be3dee2854afb2d3e4c68497364a2f27a377fc
```

> **Step 3: Sign the hash generated in Step 2 using the privateKey.**

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

> **Step 4: Combine all parameters along with the signature generated in Step 3 into the request body.**

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

## Example of POST /fapi/v3/order

#### Example: All parameters are sent through the query string (Python 3.9.6).

#### Example: The following parameters are API registration information. user, signer, and privateKey are for demonstration purposes only (where privateKey is the private key of the agent).

| Key        | Value                                                              |
| ---------- | ------------------------------------------------------------------ |
| user       | 0x63DD5aCC6b1aa0f563956C0e534DD30B6dcF7C4e                         |
| signer     | 0x21cF8Ae13Bb72632562c6Fff438652Ba1a151bb0                         |
| privateKey | 0x4fd0a42218f3eae43a6ce26d22544e986139a01e5b34a62db53757ffca81bae1 |

#### Example: The nonce parameter should be the current system timestamp in microseconds. If it exceeds the current time or lags behind the system time by more than 5 seconds, it will be considered an invalid request.

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

#### Example: The following parameters are business request parameters.

```python
my_dict = {'symbol':'SANDUSDT','side':"SELL","type":'LIMIT','orderId':2194215}
```

> **Step 1: Convert all business parameters to strings and generate a sorted string based on ASCII order.**

```python
    #Define all element values as strings.
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


    #Remove elements with empty values.
    my_dict = {key: value for key, value in my_dict.items() if  value is not None}
    my_dict['recvWindow'] = 50000
    my_dict['timestamp'] = int(round(time.time()*1000))
    # my_dict['timestamp'] = 1749545309665
	#Convert all elements to strings.
    _trim_dict(my_dict)
    #Generate a string sorted by ASCII values and remove special characters.
    json_str = json.dumps(my_dict, sort_keys=True).replace(' ', '').replace('\'','\"')
    print(json_str)
    #{"orderId":"2194215","recvWindow":"50000","side":"BUY","symbol":"SANDUSDT","timestamp":"1749545309665","type":"LIMIT"}
```

> **Step 2: Use ABI encoding to combine the string from step 1 with the account information (user, signer) and nonce, then generate a hash string from the encoded result.**

```python
   from eth_abi import encode
   from web3 import Web3

   #Use WEB3 ABI to encode the generated string together with user, signer, and nonce.
   encoded = encode(['string', 'address', 'address', 'uint256'], [json_str, user, signer, nonce])
   print(encoded.hex())
   #000000000000000000000000000000000000000000000000000000000000008000000000000000000000000063dd5acc6b1aa0f563956c0e534dd30b6dcf7c4e00000000000000000000000021cf8ae13bb72632562c6fff438652ba1a151bb00000000000000000000000000000000000000000000000000006361457bcec8300000000000000000000000000000000000000000000000000000000000000767b226f726465724964223a2232313934323135222c227265637657696e646f77223a223530303030222c2273696465223a22425559222c2273796d626f6c223a2253414e4455534454222c2274696d657374616d70223a2231373439353435333039363635222c2274797065223a224c494d4954227d00000000000000000000
   keccak_hex =Web3.keccak(encoded).hex()
   print(keccak_hex)
   #6ad9569ea1355bf62de1b09b33b267a9404239af6d9227fa59e3633edae19e2a
```

> **Step 3: Sign the hash generated in Step 2 using the privateKey.**

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

> **Step 4: Combine all parameters and the signature from Step 3 into the request body.**

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

## python script

```python
#Python 3.9.6
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

## Public Endpoints Info

### Terminology

* `base asset` refers to the asset that is the `quantity` of a symbol.
* `quote asset` refers to the asset that is the `price` of a symbol.

### ENUM definitions

**Symbol type:**

* FUTURE

**Contract type (contractType):**

* PERPETUAL

**Contract status(contractStatus，status):**

* PENDING_TRADING
* TRADING
* PRE_SETTLE
* SETTLING
* CLOSE

**Order status (status):**

* NEW
* PARTIALLY_FILLED
* FILLED
* CANCELED
* REJECTED
* EXPIRED

**Order types (orderTypes, type):**

* LIMIT
* MARKET
* STOP
* STOP_MARKET
* TAKE_PROFIT
* TAKE_PROFIT_MARKET
* TRAILING_STOP_MARKET

**Order side (side):**

* BUY
* SELL

**Position side (positionSide):**

* BOTH
* LONG
* SHORT

**Time in force (timeInForce):**

* GTC - Good Till Cancel
* IOC - Immediate or Cancel
* FOK - Fill or Kill
* GTX - Good Till Crossing	(Post Only)

**Working Type (workingType)**

* MARK_PRICE
* CONTRACT_PRICE

**Response Type (newOrderRespType)**

* ACK
* RESULT

**Kline/Candlestick chart intervals:**

m -> minutes; h -> hours; d -> days; w -> weeks; M -> months

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

**Rate limiters (rateLimitType)**

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

* REQUEST_WEIGHT
* ORDERS

**Rate limit intervals (interval)**

* MINUTE

## Filters

Filters define trading rules on a symbol or an exchange.

### Symbol filters

#### PRICE_FILTER

> **/exchangeInfo format:**

```javascript
{
    "filterType": "PRICE_FILTER",
    "minPrice": "0.00000100",
    "maxPrice": "100000.00000000",
    "tickSize": "0.00000100"
  }
```

The `PRICE_FILTER` defines the `price` rules for a symbol. There are 3 parts:

* `minPrice` defines the minimum `price`/`stopPrice` allowed; disabled on `minPrice` == 0.
* `maxPrice` defines the maximum `price`/`stopPrice` allowed; disabled on `maxPrice` == 0.
* `tickSize` defines the intervals that a `price`/`stopPrice` can be increased/decreased by; disabled on `tickSize` == 0.

Any of the above variables can be set to 0, which disables that rule in the `price filter`. In order to pass the `price filter`, the following must be true for `price`/`stopPrice` of the enabled rules:

* `price` >= `minPrice`
* `price` <= `maxPrice`
* (`price`-`minPrice`) % `tickSize` == 0

#### LOT_SIZE

> **/exchangeInfo format:**

```javascript
{
    "filterType": "LOT_SIZE",
    "minQty": "0.00100000",
    "maxQty": "100000.00000000",
    "stepSize": "0.00100000"
  }
```

The `LOT_SIZE` filter defines the `quantity` (aka "lots" in auction terms) rules for a symbol. There are 3 parts:

* `minQty` defines the minimum `quantity` allowed.
* `maxQty` defines the maximum `quantity` allowed.
* `stepSize` defines the intervals that a `quantity` can be increased/decreased by.

In order to pass the `lot size`, the following must be true for `quantity`:

* `quantity` >= `minQty`
* `quantity` <= `maxQty`
* (`quantity`-`minQty`) % `stepSize` == 0

#### MARKET_LOT_SIZE

> **/exchangeInfo format:**

```javascript
{
    "filterType": "MARKET_LOT_SIZE",
    "minQty": "0.00100000",
    "maxQty": "100000.00000000",
    "stepSize": "0.00100000"
  }
```

The `MARKET_LOT_SIZE` filter defines the `quantity` (aka "lots" in auction terms) rules for `MARKET` orders on a symbol. There are 3 parts:

* `minQty` defines the minimum `quantity` allowed.
* `maxQty` defines the maximum `quantity` allowed.
* `stepSize` defines the intervals that a `quantity` can be increased/decreased by.

In order to pass the `market lot size`, the following must be true for `quantity`:

* `quantity` >= `minQty`
* `quantity` <= `maxQty`
* (`quantity`-`minQty`) % `stepSize` == 0

#### MAX_NUM_ORDERS

> **/exchangeInfo format:**

```javascript
{
    "filterType": "MAX_NUM_ORDERS",
    "limit": 200
  }
```

The `MAX_NUM_ORDERS` filter defines the maximum number of orders an account is allowed to have open on a symbol.

Note that both "algo" orders and normal orders are counted for this filter.

#### MAX_NUM_ALGO_ORDERS

> **/exchangeInfo format:**

```javascript
{
    "filterType": "MAX_NUM_ALGO_ORDERS",
    "limit": 100
  }
```

The `MAX_NUM_ALGO_ORDERS ` filter defines the maximum number of all kinds of algo orders an account is allowed to have open on a symbol.

The algo orders include `STOP`, `STOP_MARKET`, `TAKE_PROFIT`, `TAKE_PROFIT_MARKET`, and `TRAILING_STOP_MARKET` orders.

#### PERCENT_PRICE

> **/exchangeInfo format:**

```javascript
{
    "filterType": "PERCENT_PRICE",
    "multiplierUp": "1.1500",
    "multiplierDown": "0.8500",
    "multiplierDecimal": 4
  }
```

The `PERCENT_PRICE` filter defines valid range for a price based on the mark price.

In order to pass the `percent price`, the following must be true for `price`:

* BUY: `price` <= `markPrice` * `multiplierUp`
* SELL: `price` >= `markPrice` * `multiplierDown`

#### MIN_NOTIONAL

> **/exchangeInfo format:**

```javascript
{
    "filterType": "MIN_NOTIONAL",
    "notional": "1"
  }
```

The `MIN_NOTIONAL` filter defines the minimum notional value allowed for an order on a symbol.
An order's notional value is the `price` * `quantity`.
Since `MARKET` orders have no price, the mark price is used.

---
