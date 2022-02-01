# Golang Trading Assignment

In this assignment, you will be building a trading backend in Golang that pairs and executes limit orders as well as 
exposes some
market data.

## Pairing orders

A matching engine is a technology that lies at the core of any exchange. From a high-level perspective matching engine
matches people (or organizations)
who want to buy an asset with people who want to sell an asset. We want to support only one type of order for this
exercise, namely limit order. A limit order is an order to buy an asset at no more than a specific price or sell an
asset at no less than a specific price.

In real life, different algorithms can be used to match orders. Here we expect you to implement a continuous trading
Price/Time algorithm (aka FIFO). This algorithm ensures that all orders at the same price level are filled in the order
in which they are received; the first order at a price level is the first order matched. For any new order, you
prioritize the opposite order with the best price, and if you have multiple orders with the same price, the earliest
takes precedence.

Usually, the term order book is used to list all active buy and sell orders. For example, consider such order book:

| ID | Direction | Time  | Amount | Price | Amount | Time  | Direction |
|----|-----------|-------|-------:|-------|-------:|-------|-----------|
| 3  |           |       |        | 10.05 | 40     | 07:03 | SELL      |
| 1  |           |       |        | 10.05 | 20     | 07:00 | SELL      |
| 2  |           |       |        | 10.04 | 20     | 07:02 | SELL      |
| 5  | BUY       | 07:06 | 40     | 10.02 |        |       |           |
| 4  | BUY       | 07:05 | 20     | 10.00 |        |       |           |
| 6  | BUY       | 07:10 | 40     | 10.00 |        |       |           |

If a new limit order `buy 55 shares at 10.06 price` comes in, then it will be filled in this order:

1. 20 shares at 10.04 (order 2)
2. 20 shares at 10.05 (order 1)
3. 15 shares at 10.05 (order 3)

This leaves the order book in the following state:

| ID | Direction | Time  | Amount | Price | Amount | Time  | Direction |
|----|-----------|-------|-------:|-------|-------:|-------|-----------|
| 3  |           |       |        | 10.05 | 25     | 07:03 | SELL      |
| 5  | BUY       | 07:06 | 40     | 10.02 |        |       |           |
| 4  | BUY       | 07:05 | 20     | 10.00 |        |       |           |
| 6  | BUY       | 07:10 | 40     | 10.00 |        |       |           |

NB: order 3 is executed only partially.

Your service will recieve orders from a Kafka topic `orders` with the following format:

```json
{
  "asset": "BTC",
  "price": 30000.24,
  "amount": 1,
  "direction": "BUY"
}
```

The fields are:

- `asset` - string, asset name, for simplicity this can be any text
- `price` - number, a price for limit order
- `amount` - number, amount of asset to fill by order
- `direction` - string, can be either "BUY" or "SELL"

The service should pair and execute orders as they are coming in, and maintain an order book for outstanding
orders.

## Rest API

The service should expose two endpoints per asset where real-time(ish) market data can be collected. These are:

- `GET /{asset}/depth` - provides the market depth for the asset
- `GET /{asset}/volume` - provides the trading volume for the asset

### Market Depth

The market depth is similar to the order book, but where orders of the same price are combined to show the total
quantity of an asset being offered/asked at a certain price. For instance if there are three buy orders at the same
price like this:

| ID | Direction | Time  | Amount | Price |
|----|-----------|-------|-------:|-------|
| 5  | BUY       | 07:06 | 40     | 10.00 |
| 4  | BUY       | 07:05 | 25     | 10.00 |
| 6  | BUY       | 07:10 | 40     | 10.00 |

Then the resulting entry in the market depth response for this asset would look like:
```json
{
  "bids": [
    [
      10.00, // price
      105.00 // quantity 
    ],
    ...
   ],
  ...
}
```
as  the total amount being sought at the price `10` is `105`.

A full example response would look something like this:
```json
GET /btc/depth
{
  "bids": [
    [
      30000.24, // price
      3.0       // quantity
    ],
    ...
  ],
  "asks": [
    [
      30001.28, // price
      2.5       // quantity
    ],
    ...
  ]
}
```

The fields are:

- `bids` - an array of (price, quantity) tuples formed from combining the BUY orders
- `asks` - an array of (price, quantity) tuples formed from combining the SELL orders

### Volume

The volume traded of an asset, is just the total EUR amount of all executed orders for a given asset. Normally this 
is calculated over some time period (such as 24h), however for our purposes we will just use the full market history 
of all orders received by our service.

A full example of the response looks like:
```json
GET /btc/volume
{
  "asset": "BTC",
  "volume": 100000.25
}
```

The fields are:

- `asset` - the asset identifier
- `volume` - the volume of all executed orders (in EUR)

## Requirements

- Clear and concise code
- Should demonstrate a knowledge of tools and practices in Golang (eg. think about asynchronous processes and how 
  best to handle them)
- Don't over-engineer, think about the simplest solution and implement it
