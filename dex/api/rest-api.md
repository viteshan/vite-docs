---
order: 1
---

# ViteX REST API

:::tip Notice
- New v2 ViteX API is released. The v1 API is deprecated and will stop service soon. Please kindly upgrade your client ASAP.
:::

## Overview
ViteX API enables users to complete trading operations on ViteX decentralized exchange without exposing private keys.
ViteX API is categorized into trading API and market trends API. Trading API (also known as private API) requires authentication and authorization, and provides functions such as order placement and cancellation. 
Market trends API (also known as public API) provides market data, information query, etc. Market trends API can be accessed publicly without authentication.

## Base Endpoint
* **[MainNet]**: `https://api.vitex.net/`
* **[TestNet]**: `https://api.vitex.net/test`

## API Response
API response is returned in JSON. 

HTTP code:

* HTTP `200` API returned successfully
* HTTP `4XX` Wrong API request
* HTTP `5XX` Service error

Response format:

Key | Value
------------ | ------------
code | `0` - success. An error code is returned if the API request failed
msg | Detailed error message
data | Return data

Example:
```json
{
  "code": 1,
  "msg": "Invalid symbol",
  "data": {}
}
```

Error code: 
* `0` API returned successfully
* `1` General error - view the specific error message in `msg` field.
* `1001` Too frequent request - request exceeds limit. 
* `1002` Invalid parameter - this may include invalid timestamp, wrong order price, invalid amount, order too small, invalid market, insufficient permission, symbol not exist, etc.
* `1003` Network - network jam, network broken, insufficient quota and so on.
* `1004` Other failure - such as attempting to cancel an order of other address, attempting to cancel an order already filled, order status exception
* `1005` Service error - unexpected API error
* `1006` Minimum order quantity not satisfied - order quantity doesn't reach the minimal requirement in the market
* `1007` Insufficient Exchange Balance - user's balance in the exchange is not enough

## Data Definition

### Order Status
Code | Status | Description
------------ | ------------ | ------------
0 | Unknown | Status unknown
1 | Pending Request | Order submitted. A corresponding request transaction has been created on the blockchain
2 | Received | Order received by ViteX smart contract. Not yet dispatched into matching engine
3 | Open | Order unfilled
4 | Filled | Order completely filled
5 | Partially Filled | Order partially filled
6 | Pending Cancel | Cancel order request submitted. A corresponding request transaction has been created on the blockchain
7 | Cancelled | Order cancelled
8 | Partially Cancelled | Order partially cancelled (order is partially filled and then cancelled)
9 | Failed | Order failed
10 | Expired | Order expired

### Order Type

Code | Status | Description
------------ | ------------ | ------------
0 | Limit Order | Limit Order
1 | Market Order | Market Order (not supported at present)

### Side

Code | Status | Description
------------ | ------------ | ------------
0 | Buy Order | Buy
1 | Sell Order | Sell

### Time In Force

Code | Status | Description
------------ | ------------ | ------------
0 | GTC - Good till Cancel | Place an order and wait for it to be fully filled or cancelled
1 | IOC - Immediate or Cancel | Place an order and immediately cancel unfilled (not supported at present)
2 | FOK - Fill or Kill | Place an order only when it can be fully filled (not supported at present)

## Private API Authorization

To use ViteX Private REST API, you must authorize at [Trade Delegation](https://x.vite.net/tradeTrust) on ViteX platform first to authorize ViteX API service to trade on behalf of you. You should fill in the delegation address, which is generated by the API service when you applied the API Key. You DO NOT need provide private key or mnemonic phrase.

* By providing **API Key** and **API Secret** to a trustworthy third party market maker (instead of private key or mnemonics), your fund is safe in your ViteX account and cannot be misappropriated. 
* You should enable the API on selected trading pairs explicitly. Attempting to trade on unauthorized pairs will cause error. 
* Authorization can be canceled at any time. In this case, even though the API Key and API Secret are still valid, ViteX exchange will reject API trading request eventually. 

It's highly recommended to enable API authorization ONLY on specific trading pairs that you wish to trade.  

:::tip Delegation Address and Quota
ViteX API service will generate a unique delegation address for each user. Orders placed by the API are signed by delegation address instead of your private key. Therefore, **DO NOT give your private key or account mnemonics to anyone**.

Meanwhile, quota of delegation address is zero by default. It's your responsibility to provide quota to the address.
:::

### Trigger Limit
ViteX Private API has a trigger limit. The limit is reset in every cycle (60s). When the limit is reached, subsequent API requests in the cycle will be rejected.  

### API Authentication

Private API requires signature authentication by **API Key** and **API Secret**, which you can apply for at [API](https://x.vite.net/tradeOpenapi) on ViteX platform. Please note that API Key and API Secret are both case sensitive.

Besides parameters defined by specific API methods, 3 additional parameters `key`, `timestamp` and `signature` should also be included. 

* `key` - Your **API Key**
* `timestamp` - UNIX timestamp in milliseconds. To avoid replay attack, API request will be rejected if the timestamp in request is **5,000 ms** earlier or **1,000 ms** later than standard time.  
* `signature` - HMAC SHA256 signature on request string, using **API Secret** as secret key

Sample code of timestamp checking at server side:

```java
    if (timestamp < (serverTime + 1000) && (serverTime - timestamp) <= 5000) {
        // process request
    } else {
        // reject request
    }
```

### Signature of Request String

* List all parameters (including `key` and `timestamp`) in alphabet order;
* Generate request string by concatenating parameters with `=` and `&` in above order;
* Sign the request string by HMAC SHA256, using **API Secret** as secret key. If request string and request body are both present, put request string in ahead of request body;
* Signature is case in-sensitive;
* Attach the signature to request string in `signature` field.

### An Example

Let's place an order through API `/api/v2/order`. Assume we have the following API Key and Secret:

API Key | API Secret
------------ | ------------
913423DE46E97751CCC734F018F09217 | F6BED9F34912C0B658B58C73B6531721

We place an order on market ETH-000_BTC-000 to buy 10 ETH at price 0.09 BTC. The API request has the following parameters:

Key | Value
------------ | ------------
symbol | ETH-000_BTC-000
side | 0
amount | 10
price | 0.09
timestamp | 1567067137937

The request string is:

`amount=10&key=913423DE46E97751CCC734F018F09217&price=0.09&side=0&symbol=ETH-000_BTC-000&timestamp=1567755178560`

Create signature:

```bash
$ echo -n "amount=10&key=913423DE46E97751CCC734F018F09217&price=0.09&side=0&symbol=ETH-000_BTC-000&timestamp=1567755178560" | openssl dgst -sha256 -hmac "F6BED9F34912C0B658B58C73B6531721"
(stdin)= 47033f6086b2afc54a0a1f837dccb4ea6fe56a0312bab311e0bbc3941efaf8d6
```

Call API to place the order:

```bash
$ curl -X POST -d "amount=10&key=913423DE46E97751CCC734F018F09217&price=0.09&side=0&symbol=ETH-000_BTC-000&timestamp=1567755178560&signature=47033f6086b2afc54a0a1f837dccb4ea6fe56a0312bab311e0bbc3941efaf8d6" https://api.vitex.net/test/api/v2/order/test
```

## Private REST API

### Place Order (test)
```
POST /api/v2/order/test
```
Test placing order. The request will not be submitted to exchange. This API is generally used to verify that the signature is correct.

**Quota consumption:**
0 UT

**Parameter:**

Name | Type | Is Required? | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES | Trading pair name. For example, `ETH-000_BTC-000`
amount | STRING | YES | Order amount (in trade token)
price | STRING | YES | Order price
side | INT | YES | Buy - `0`, Sell - `1`
timestamp | LONG | YES | Timestamp (s)
key | STRING | YES | API Key
signature | STRING | YES | HMAC SHA256 signature of request string

**Response:**

```json
{
  "code": 0,
  "msg": "ok",   
  "data": null
}
```

### Place Order
```
POST /api/v2/order
```

**Quota consumption:**
1 UT

**Parameter:**

Name | Type | Is Required? | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES | Trading pair name. For example, `ETH-000_BTC-000`
amount | STRING | YES | Order amount (in trade token)
price | STRING | YES | Order price
side | INT | YES | Buy - `0`, Sell - `1`
timestamp | LONG | YES | Timestamp (s)
key | STRING | YES | API Key
signature | STRING | YES | HMAC SHA256 signature of request string

**Response:**

Name | Type | Description
------------ | ------------ | ------------
symbol | STRING | Trading pair name
orderId | STRING | Order ID
status | INTEGER | Order status

```json
{
  "code": 0,
  "msg": "ok",
  "data": {
    "symbol": "VX_ETH-000",
    "orderId": "c35dd9868ea761b22fc76ba35cf8357db212736ecb56399523126c515113f19d",
    "status": 1
  }
}
```
### Cancel Order
```
DELETE /api/v2/order
```

**Quota consumption:**
1 UT

**Parameter:**

Name | Type | Is Required? | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES | Trading pair name. For example, `ETH-000_BTC-000`
orderId | STRING | YES | Order ID
timestamp | LONG | YES | Timestamp (s)
key | STRING | YES | API Key
signature | STRING | YES | HMAC SHA256 signature of request string

**Response:**

Name | Type | Description
------------ | ------------ | ------------
symbol | STRING | Trading pair name
orderId | STRING | Order ID
cancelRequest | STRING | Cancel request ID
status | INTEGER | Order status

```json
{
  "code": 0,
  "msg": "ok",
  "data": {
    "symbol": "VX_ETH-000",
    "orderId": "c35dd9868ea761b22fc76ba35cf8357db212736ecb56399523126c515113f19d",
    "cancelRequest": "2d015156738071709b11e8d6fa5a700c2fd30b28d53aa6160fd2ac2e573c7595",
    "status": 6
  }
}
```

### Cancel All Orders
```
DELETE /api/v2/orders
```

**Quota consumption:**
N UT (N=Orders)

**Parameters:**

Name | Type | Is Required? | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES | Trading pair name. For example, `ETH-000_BTC-000`
timestamp | LONG | YES | Timestamp (s)
key | STRING | YES | API Key
signature | STRING | YES | HMAC SHA256 signature of request string

**Response:**

Name | Type | Description
------------ | ------------ | ------------
symbol | STRING | Trading pair name
orderId | STRING | Order ID
cancelRequest | STRING | Cancel request ID
status | INTEGER | Order status

```json
{
  "code": 0,
  "msg": "ok",
  "data": [
    {
      "symbol": "VX_ETH-000",
      "orderId": "de185edae25a60dff421c1be23ac298b121cb8bebeff2ecb25807ce7d72cf622",
      "cancelRequest": "355b6fab007d86e7ff09b0793fbb205e82d3880b64d948ed46f88237115349ab",
      "status": 6
    },
    {
      "symbol": "VX_ETH-000",
      "orderId": "7e079d4664791207e082c0fbeee7b254f2a31e87e1cff9ba18c5faaeee3d400a",
      "cancelRequest": "55b80fe42c41fa91f675c04a8423afa85857cd30c0f8878d52773f7096bfac3b",
      "status": 6
    }
  ]
}
```

## Public REST API

### Get Order Limit
```
GET /api/v2/limit
```
Get minimum order quantity for all markets

* **Response:**

  :::demo
  ```json tab:Response
  {
    "code": 0,
    "msg": "ok",
    "data": {
        "minAmount": {
            "BTC-000": "0.0001",
            "USDT-000": "1",
            "ETH-000": "0.01"
        },
        "depthStepsLimit": {}
    }
  }
  ```
  ```json test: "Test" url: /api/v2/limit method: GET
  {}
  ```
  :::

### Get All Tokens
```
GET /api/v2/tokens
```

* **Parameters:**

Name | Type | Is Required? | Description
------------ | ------------ | ------------ | ------------
category | STRING | NO | Token category, [`quote`,`all`], default `all`
tokenSymbolLike | STRING | NO | Token symbol. For example, `VITE`. Fuzzy search supported.
offset | INTEGER | NO | Search starting index, starts at `0`, default `0`
limit | INTEGER | NO | Search limit, max `500`, default `500`

* **Response:**

  :::demo
  
  ```json tab:Response
  {
    "code": 0,
    "msg": "ok",
    "data": [
      {
        "tokenId": "tti_322862b3f8edae3b02b110b1",
        "name": "BTC Token",
        "symbol": "BTC-000",
        "originalSymbol": "BTC",
        "totalSupply": "2100000000000000",
        "owner": "vite_ab24ef68b84e642c0ddca06beec81c9acb1977bbd7da27a87a",
        "tokenDecimals": 8,
        "urlIcon": null
      }
    ]
  }
  ```
  
  ```json test:Test url: /api/v2/tokens?tokenSymbolLike=ETH method: GET
  {}
  ```
  :::

### Get Token Detail
```
GET /api/v2/token/detail
```

* **Parameters:**

Name | Type | Is Required? | Description
------------ | ------------ | ------------ | ------------
tokenSymbol | STRING | NO | Token symbol. For example, `VITE`
tokenId | STRING | NO | Token id. For example, `tti_5649544520544f4b454e6e40`

* **Response:**

  :::demo
  
  ```json tab:Response
  {
    "code": 0,
    "msg": "ok",
    "data": {
      "tokenId": "tti_322862b3f8edae3b02b110b1",
      "name": "BTC Token",
      "symbol": "BTC-000",
      "originalSymbol": "BTC",
      "totalSupply": "2100000000000000",
      "publisher": "vite_ab24ef68b84e642c0ddca06beec81c9acb1977bbd7da27a87a",
      "tokenDecimals": 8,
      "tokenAccuracy": "0.00000001",
      "publisherDate": null,
      "reissue": 2,
      "urlIcon": null,
      "gateway": null,
      "website": null,
      "links": null,
      "overview": null
    }
  }
  ```
  
  ```json test:Test url: /api/v2/token/detail?tokenId=tti_5649544520544f4b454e6e40 method: GET
  {}
  ```
  :::
  
### Get Listed Tokens
```
GET /api/v2/token/mapped
```
Get tokens that are already listed in specific market

* **Parameters:**

Name | Type | Is Required? | Description
------------ | ------------ | ------------ | ------------
quoteTokenSymbol | STRING | YES | Quote token symbol. For example, `VITE`

* **Response:**

  :::demo
  
  ```json tab:Response
  {
    "code": 0,
    "msg": "ok",
    "data": [
      {
        "tokenId": "tti_c2695839043cf966f370ac84",
        "symbol": "VCP"
      }
    ]
  }
  ```
  
  ```json test:Test url: /api/v2/token/mapped?quoteTokenSymbol=VITE method: GET
  {}
  ```
  :::
  
### Get Unlisted Tokens
```
GET /api/v2/token/unmapped
```
Get tokens that are not yet listed in specific market

* **Parameters:**

Name | Type | Is Required? | Description
------------ | ------------ | ------------ | ------------
quoteTokenSymbol | STRING | YES | Quote token symbol. For example, `VITE`

* **Response:**

  :::demo
  
  ```json tab:Response
  {
    "code": 0,
    "msg": "ok",
    "data": [
      {
        "tokenId": "tti_2736f320d7ed1c2871af1d9d",
        "symbol": "VTT"
      }
    ]
  }
  ```
  
  ```json test:Test url: /api/v2/token/unmapped?quoteTokenSymbol=VITE method: GET
  {}
  ```
  :::
  
### Get Trading Pair
```  
GET /api/v2/market
```
Get trading pair in detail

* **Parameters:**

Name | Type | Is Required? | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES | Trading pair name. For example, `GRIN-000_BTC-000`

* **Response：**

  :::demo
  
  ```json tab:Response
  {
     "code": 0,
        "msg": "ok",
        "data": {
            "symbol": "GRIN-000_BTC-000",
            "tradingCurrency": "GRIN-000",
            "quoteCurrency": "BTC-000",
            "tradingCurrencyId": "tti_289ee0569c7d3d75eac1b100",
            "quoteCurrencyId": "tti_b90c9baffffc9dae58d1f33f",
            "tradingCurrencyName": "Grin",
            "quoteCurrencyName": "Bitcoin",
            "operator": "vite_4c2c19f563187163145ab8f53f5bd36864756996e47a767ebe",
            "operatorName": "Vite Labs",
            "operatorLogo": "https://token-profile-1257137467.cos.ap-hongkong.myqcloud.com/icon/f62f3868f3cbb74e5ece8d5a4723abef.png",
            "pricePrecision": 8,
            "amountPrecision": 2,
            "minOrderSize": "0.0001",
            "operatorMakerFee": 5.0E-4,
            "operatorTakerFee": 5.0E-4,
            "highPrice": "0.00007000",
            "lowPrice": "0.00006510",
            "lastPrice": "0.00006682",
            "volume": "1476.37000000",
            "baseVolume": "0.09863671",
            "bidPrice": "0.00006500",
            "askPrice": "0.00006999",
            "openBuyOrders": 27,
            "openSellOrders": 42
        }
  }
  ```
  
  ```json test:Test url: /api/v2/market?symbol=GRIN-000_BTC-000 method: GET
  {}
  ```
  :::
 
### Get All Trading Pairs
```
GET /api/v2/markets
```

* **Parameters:**

Name | Type | Is Required? | Description
------------ | ------------ | ------------ | ------------
offset | INTEGER | NO | Search starting index, starts at `0`, default `0`
limit | INTEGER | NO | Search limit, max `500`, default `500`

* **Response:**

  :::demo
  
  ```json tab:Response
  {
    "code": 0,
    "msg": "ok",
    "data": [
      {
        "symbol": "BTC-000_USDT",
        "tradeTokenSymbol": "BTC-000",
        "quoteTokenSymbol": "USDT-000",
        "tradeToken": "tti_322862b3f8edae3b02b110b1",
        "quoteToken": "tti_973afc9ffd18c4679de42e93",
        "pricePrecision": 8,
        "quantityPrecision": 8
      }
    ]
  }
  ```
  
  ```json test:Test url: /api/v2/markets method: GET
  {}
  ```
  :::

### Get Order
```
GET /api/v2/order
```

* **Parameters:**

Name | Type | Is Required? | Description
------------ | ------------ | ------------ | ------------
address | STRING | YES | User's account address (not delegation address)
orderId | STRING | YES | Order id

* **Response:**

  :::demo
  
  ```json tab:Response
  {
    "code": 0,
    "msg": "ok",
    "data": {
      "address": "vite_228f578d58842437fb52104b25750aa84a6f8558b6d9e970b1",
      "orderId": "0dfbafac33fbccf5c65d44d5d80ca0b73bc82ae0bbbe8a4d0ce536d340738e93",
      "symbol": "VX_ETH-000",
      "tradeTokenSymbol": "VX",
      "quoteTokenSymbol": "ETH-000",
      "tradeToken": "tti_564954455820434f494e69b5",
      "quoteToken": "tti_06822f8d096ecdf9356b666c",
      "side": 1,
      "price": "0.000228",
      "quantity": "100.0001",
      "amount": "0.02280002",
      "executedQuantity": "100.0000",
      "executedAmount": "0.022800",
      "executedPercent": "0.999999",
      "executedAvgPrice": "0.000228",
      "fee": "0.000045",
      "status": 5,
      "type": 0,
      "createTime": 1586941713
    }
  }
  ```
  
  ```json test:Test url: /api/v2/order?address=vite_ff38174de69ddc63b2e05402e5c67c356d7d17e819a0ffadee method: GET
  {}
  ```
  :::  

### Get Open Order
```
GET /api/v2/orders/open
```
Get orders that are unfilled or partially filled.

* **Parameters:**

Name | Type | Is Required? | Description
------------ | ------------ | ------------ | ------------
address | STRING | YES | User's account address (not delegation address)
symbol | STRING | NO | Trading pair name. For example, `GRIN-000_BTC-000`
quoteTokenSymbol | STRING | NO | Quote token symbol. For example, `BTC-000`
tradeTokenSymbol | STRING | NO | Trade token symbol. For example, `GRIN-000`
offset | INTEGER | NO | Search starting index, starts at `0`, default `0`
limit | INTEGER | NO | Search limit, default `30`, max `100`
total | INTEGER | NO | Include total number searched in result? `0` - not included, `1` - included. Default is `0`, in this case `total=-1` in response

* **Response:**

  :::demo
  ```json tab:Response
  {
    "code": 0,
    "msg": "ok",
    "data": {
      "order": [
        {
          "address": "vite_ff38174de69ddc63b2e05402e5c67c356d7d17e819a0ffadee",
          "orderId": "5379b281583bb17c61bcfb1e523b95a6c153150e03ce9db35f37d652bbb1b321",
          "symbol": "BTC-000_USDT-000",
          "tradeTokenSymbol": "BTC-000",
          "quoteTokenSymbol": "USDT-000",
          "tradeToken": "tti_322862b3f8edae3b02b110b1",
          "quoteToken": "tti_973afc9ffd18c4679de42e93",
          "side": 0,
          "price": "1.2000",
          "quantity": "1.0000",
          "amount": "1.20000000",
          "executedQuantity": "0.0000",
          "executedAmount": "0.0000",
          "executedPercent": "0.0000",
          "executedAvgPrice": "0.0000",
          "confirmations": null,
          "fee": "0.0000",
          "status": 3,
          "type": 0,
          "createTime": 1587906622
        }
      ]
    }
  }
  ```
  ```json test: "Test" url: /api/v2/orders/open?address=vite_ff38174de69ddc63b2e05402e5c67c356d7d17e819a0ffadee method: GET
  {}
  ```
  :::

### Get Orders
```
GET /api/v2/orders
```

* **Parameters:**

Name | Type | Is Required? | Description
------------ | ------------ | ------------ | ------------
address | STRING | YES | User's account address (not delegation address)
symbol | STRING | NO | Trading pair name. For example, `GRIN-000_BTC-000`
quoteTokenSymbol | STRING | NO | Quote token symbol. For example, `BTC-000`
tradeTokenSymbol | STRING | NO | Trade token symbol. For example, `GRIN-000`
startTime | LONG | NO | Start time (s)
endTime | LONG | NO | End time (s)
side | INTEGER | NO | Order side. `0` - buy, `1` - sell
status | INTEGER | NO | Order status, valid in [`0-10`]. `3`,`5` - returns orders that are unfilled or partially filled; `7`,`8` - returns orders that are cancelled or partially cancelled
offset | INTEGER | NO | Search starting index, starts at `0`, default `0`
limit | INTEGER | NO | Search limit, default `30`, max `100`
total | INTEGER | NO | Include total number searched in result? `0` - not included, `1` - included. Default is `0`, in this case `total=-1` in response

* **Response:**

  :::demo
  
  ```json tab:Response
  {
    "code": 0,
    "msg": "ok",
    "data": {
      "order": [
        {
          "address": "vite_ff38174de69ddc63b2e05402e5c67c356d7d17e819a0ffadee",
          "orderId": "0dfbafac33fbccf5c65d44d5d80ca0b73bc82ae0bbbe8a4d0ce536d340738e93",
          "symbol": "VX_ETH-000",
          "tradeTokenSymbol": "VX",
          "quoteTokenSymbol": "ETH-000",
          "tradeToken": "tti_564954455820434f494e69b5",
          "quoteToken": "tti_06822f8d096ecdf9356b666c",
          "side": 1,
          "price": "0.000228",
          "quantity": "100.0001",
          "amount": "0.02280002",
          "executedQuantity": "100.0000",
          "executedAmount": "0.022800",
          "executedPercent": "0.999999",
          "executedAvgPrice": "0.000228",
          "fee": "0.000045",
          "status": 5,
          "type": 0,
          "createTime": 1586941713
        }
      ],
      "total": -1
    }
  }
  ```
  
  ```json test:Test url: /api/v2/orders?address=vite_ff38174de69ddc63b2e05402e5c67c356d7d17e819a0ffadee method: GET
  {}
  ```
  :::

### Get 24hr Ticker Price Changes
```
GET /api/v2/ticker/24hr
```

* **Parameters:**

Name | Type | Is Required? | Description
------------ | ------------ | ------------ | ------------
symbols | STRING | NO | Trading pairs, split by ","
quoteTokenSymbol | STRING | NO | Quote token symbol. For example, `USDT-000`. Returns all pairs if not present

* **Response:**

  :::demo
  
  ```json tab:Response
  {
    "code": 0,
    "msg": "ok",
    "data": [
      {
        "symbol":"BTC-000_USDT-000",
        "tradeTokenSymbol":"BTC-000",
        "quoteTokenSymbol":"USDT-000",
        "tradeToken":"tti_b90c9baffffc9dae58d1f33f",
        "quoteToken":"tti_80f3751485e4e83456059473",
        "openPrice":"7540.0000",
        "prevClosePrice":"7717.0710",
        "closePrice":"7683.8816",
        "priceChange":"143.8816",
        "priceChangePercent":0.01908244,
        "highPrice":"7775.0000",
        "lowPrice":"7499.5344",
        "quantity":"13.8095",
        "amount":"104909.3499",
        "pricePrecision":4,
        "quantityPrecision":4,
        "openTime":null,
        "closeTime":null
      }
    ]
  }
  ```
  
  ```json test:Test url: /api/v2/ticker/24hr?quoteTokenSymbol=VITE method: GET
  {}
  ```
  :::

### Get Order Book Ticker
```  
GET /api/v2/ticker/bookTicker
```
Get current best price/qty on the order book for a trading pair

* **Parameters:**

Name | Type | Is Required? | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES | Trading pair name. For example, `GRIN-000_VITE`

* **Response:**

  :::demo
  
  ```json tab:Response
  {
    "code": 0,
    "msg": "ok",
    "data": {
      "symbol": "BTC-000_USDT-000",
      "bidPrice": "7600.0000",
      "bidQuantity": "0.7039",
      "askPrice": "7725.0000",
      "askQuantity": "0.0001",
      "height": null
    }
  }
  ```
  
  ```json test:Test url: /api/v2/ticker/bookTicker?symbol=BTC-000_VITE-000 method: GET
  {}
  ```
  :::
 
### Get Trade Summary
```
GET /api/v2/trades
```
Get trade records in summary

* **Parameters:**

Name | Type | Is Required? | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES | Trading pair name. For example.`GRIN-000_VITE`
limit | INTEGER | NO | Search limit, default `500`

* **Response：**

  :::demo
  
  ```json tab:Response
  {
    "code": 0,
    "msg": "ok",
    "data": [
      {
          "timestamp": 1588214534000,
          "price": "0.024933",
          "amount": "0.0180",
          "side": 0
      },
      {
          "timestamp": 1588214364000,
          "price": "0.024535",
          "amount": "0.0127",
          "side": 0
      }
    ]
  }
  ```
  
  ```json test:Test url: /api/v2/trades?symbol=BTC-000_USDT-000 method: GET
  {}
  ```
  :::
 

### Get Trade Records
```
Get /api/v2/trades/all
```
Get trade records in detail

* **Parameters:**

Name | Type | Is Required? | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES | Trading pair name. For example, `GRIN-000_VITE`
orderId | STRING | NO | Order id
startTime | LONG | NO | Start time (s)
endTime | LONG | NO | End time (s)
side | INTEGER | NO | Order side. `0` - buy, `1` - sell
offset | INTEGER | NO | Search starting index, starts at `0`, default `0`
limit | INTEGER | NO | Search limit, default `30`, max `100`
total | INTEGER | NO | Include total number searched in result? `0` - not included, `1` - included. Default is `0`, in this case `total=-1` in response

* **Response:**

  :::demo
  
  ```json tab:Response
  {
    "code": 0,
    "msg": "ok",
    "data": {
      "height": null,
      "trade": [
        {
          "tradeId": "d3e7529de05e94d247a4e7ef58a56b069b059d52",
          "symbol": "VX_ETH-000",
          "tradeTokenSymbol": "VX",
          "quoteTokenSymbol": "ETH-000",
          "tradeToken": "tti_564954455820434f494e69b5",
          "quoteToken": "tti_06822f8d096ecdf9356b666c",
          "price": "0.000228",
          "quantity": "0.0001",
          "amount": "0.00000002",
          "time": 1586944732,
          "side": 0,
          "buyFee": "0.00000000",
          "sellFee": "0.00000000",
          "blockHeight": 260
        }
      ],
      "total": -1
    }
  }
  ```
  
  ```json test:Test url: /api/v2/trades?symbol=BTC-000_USDT-000 method: GET
  {}
  ```
  :::
  
### Get Order Book Depth
```
GET /api/v2/depth
```

* **Parameters:**

Name | Type | Is Required? | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES | Trading pair name. For example, `GRIN-000_VITE`
limit | INTEGER | NO | Search limit, max `100`, default `100`
precision | INTEGER | NO | Price Precision

* **Response:**

  :::demo
  
  ```json tab:Response
  {
      "code": 0,
      "msg": "ok",
      "data": {
        "timestamp": 1588170501936,
        "asks": [
          [
              "0.025750",
              "0.0323"
          ],
          [
              "0.026117",
              "0.0031"
          ]    
        ],
        "bids": [
          [
              "0.024820",
              "0.0004"
          ],
          [
              "0.024161",
              "0.0042"
          ]
        ]
      }
    }
  ```
  
  ```json test:Test url: /api/v2/depth?symbol=BTC-000_USDT-000 method: GET
  {}
  ```
  :::

### Get Klines/Candlestick bars
```
GET /api/v2/klines
```

* **Parameters:**

Name | Type | Is Required? | Description
------------ | ------------ | ------------ | ------------
symbol | STRING | YES | Trading pair name. For example, `GRIN-000_VITE`
interval | STRING | YES | Interval, [`minute`, `hour`, `day`, `minute30`, `hour6`, `hour12`, `week`]
limit | INTEGER | NO | Search limit, max `1500`, default `500`
startTime | LONG | NO | Start time (s)
endTime | LONG | NO | End time (s)

* **Response:**

Name | Type | Description
------------ | ------------ | ------------
t | LONG | Timestamp
c | STRING | Close price
p | STRING | Open price
h | STRING | Highest price
l | STRING | Lowest price
v | STRING | Trade volume

  :::demo
  
  ```json tab:Response
  {
    "code": 0,
    "msg": "ok",
    "data": {
      "t": [
        1554207060
      ],
      "c": [
        1.0
      ],
      "p": [
        1.0
      ],
      "h": [
        1.0
      ],
      "l": [
        1.0
      ],
      "v": [
        12970.8
      ]
    }
  }
  ```
  
  ```json test:Test url: /api/v2/klines?symbol=VITE_BTC-000&interval=minute method: GET
  {}
  ```
  :::
  
### Get Deposit-Withdrawal Records
```
/api/v2/deposit-withdraw
```

* **Parameters:**

Name | Type | Is Required? | Description
------------ | ------------ | ------------ | ------------
address | STRING | YES | Account address
tokenId | STRING | YES | Token id. For example, `tti_5649544520544f4b454e6e40`
offset | INTEGER | NO | Search starting index, starts at `0`, default `0`
limit | INTEGER | NO | Search limit, max `100`, default `100`

* **Response:**

  :::demo
  
  ```json tab:Response
  {
    "code": 0,
    "msg": "ok",
    "data": {
      "record": [
        {
          "time": 1555057049,
          "tokenSymbol": "VITE",
          "amount": "1000000.00000000",
          "type": 1
        }
      ],
      "total": 16
    }
  }
  ```
  
  ```json test:Test url: /api/v2/deposit-withdraw?address=vite_ff38174de69ddc63b2e05402e5c67c356d7d17e819a0ffadee&tokenId=tti_5649544520544f4b454e6e40 method: GET
  {}
  ```
  :::
  
### Get Exchange Rate
```
GET /api/v2/exchange-rate
```

* **Parameters:**

Name | Type | Is Required? | Description
------------ | ------------ | ------------ | ------------
tokenSymbols | STRING | NO | Trading pairs, split by ",". For example, `VITE,ETH-000`
tokenIds | STRING | NO | Token ids, split by ",". For example, `tti_5649544520544f4b454e6e40,tti_5649544520544f4b454e6e40`

* **Response:**

  :::demo
  
  ```json tab:Response
  {
    "code": 0,
    "msg": "ok",
    "data": [
      {
        "tokenId": "tti_5649544520544f4b454e6e40",
        "tokenSymbol": "VITE",
        "usdRate": 0.03,
        "cnyRate": 0.16
      }
    ]
  }
  ```
  
  ```json test:Test url: /api/v2/exchange-rate?tokenIds=tti_5649544520544f4b454e6e40 method: GET
  {}
  ```
  ::: 

### Get USD-CNY Rate
```
GET /api/v2/usd-cny
```

* **Parameters:**
  None

* **Response:**

  :::demo
  
  ```json tab:Response
  {
    "code": 0,
    "msg": "ok",
    "data": 6.849
  }
  ```
  
  ```json test:Run url: /api/v2/usd-cny method: GET
  {}
  ```
  :::

### Get Exchange Balance
```
/api/v2/balance
```

**Parameters:**

Name | Type | Is Required? | Description
------------ | ------------ | ------------ | ------------
address | STRING | YES | Account address

**Response:**

Name | Type | Description
------------ | ------------ | ------------
available | STRING | Available balance
locked | STRING | Balance locked by open order

  :::demo
  
  ```json tab:Response
  {
    "code": 0,
    "msg": "ok",
    "data": {
      "VX": {
        "available": "0.00000000",
        "locked": "0.00000000"
      },
      "VCP": {
        "available": "373437.00000000",
        "locked": "0.00000000"
      },
      "BTC-000": {
        "available": "0.02597393",
        "locked": "0.13721639"
      },
      "USDT-000": {
        "available": "162.58284100",
        "locked": "170.40459600"
      },
      "GRIN-000": {
        "available": "0.00000000",
        "locked": "0.00000000"
      },
      "VITE": {
        "available": "30047.62090072",
        "locked": "691284.75633290"
      },
      "ETH-000": {
        "available": "1.79366977",
        "locked": "7.93630000"
      }
    }
  }
  ```
  
  ```json test:Test url: /api/v2/balance method: GET
  {}
  ```
  ::: 
  
### Get Trade Mining Info
Get the current cycle's trade mining pool size and real-time fees accumulated 
```
GET /api/v2/trade_fee_info
```
**Response:**

Name | Type | Description
------------ | ------------ | ------------
tradePoolVx | OBJECT | Minable VX to be distributed in 4 markets at the cycle end
tradePoolFee | OBJECT | Real-time cumulative trading fees in 4 markets

1: VITE
2: ETH-000
3: BTC-000
4: USDT-000

  :::demo
  
  ```json tab:Response
  {
      "code": 0,
      "msg": "ok",
      "data": {
          "tradePoolVx": {
              "1": "6005.736536774939954301",
              "2": "6005.736536774939954301",
              "3": "6005.736536774939954301",
              "4": "6005.736536774939954301"
          },
          "tradePoolFee": {
              "1": "17769.748909914626699977",
              "2": "1.267967346417481080",
              "3": "0.03045706",
              "4": "299.338260"
          }
      }
  }
  ```
  
  ```json test:Test url: /api/v2/trade_fee_info method: GET
  {}
  ```
  ::: 

### Get Server Time
```
GET /api/v2/time
GET /api/v2/timestamp
```

* **Parameters:**
  None

* **Response:**

  :::demo
  
  ```json tab:Response
  {
    "code": 0,
    "msg": "ok",
    "data": 1559033445000
  }
  ```
  
  ```json test:Run url: /api/v2/time method: GET
  {}
  ```
  ::: 
