# Merchant API Documentation

## Overview

The Merchant API allows authorized third-party fulfillment partners to find orders that need Robux delivery and report when they've completed a fulfillment transaction. This enables a distributed model for order fulfillment where multiple merchants can work together to deliver Robux efficiently.

## Authentication

All API endpoints require authentication with an API key. Include the API key in the request header:

```
X-API-Key: YOUR_API_KEY
```

Alternatively, for testing only, you can include the API key as a query parameter:

```
?apiKey=YOUR_API_KEY
```

## Getting API Keys

API keys must be created by an administrator. Contact the system administrator to request an API key for your merchant account.

## Endpoints

### Find Fulfillment Orders

```
GET /api/merchant/orders/fulfillment
```

Find orders that need fulfillment based on various criteria. This endpoint allows you to find orders with specific Robux amounts needed.

#### Query Parameters

- `robux_amount` (optional): Find orders with a developer product of exactly this Robux amount
- `min_amount` (optional): Minimum Robux amount remaining for fulfillment
- `max_amount` (optional): Maximum Robux amount remaining for fulfillment
- `limit` (optional): Maximum number of orders to return (default: 1)

#### Response

```json
{
  "success": true,
  "count": 1,
  "data": [
    {
      "id": "6123456789abcdef12345678",
      "reference": "ORD-123456-ABCDEF",
      "username": "ExampleUser",
      "gameId": "1234567890",
      "gameName": "Example Game",
      "robuxAmount": 1000,
      "amountRemaining": 600,
      "fulfillmentPercentage": 40,
      "status": "paid",
      "fulfillmentStatus": "partial",
      "createdAt": "2023-01-01T00:00:00.000Z",
      "devProducts": [
        {
          "value": 200,
          "count": 3,
          "total": 600
        },
        {
          "value": 400,
          "count": 1,
          "total": 400
        }
      ],
      "suggestedFulfillment": 200
    }
  ]
}
```

### Reserve a Specific Amount

```
POST /api/merchant/orders/:orderId/reserve
```

Reserve a specific developer product amount for fulfillment. This prevents other merchants from trying to fulfill the same amount at the same time, reducing race conditions.

#### URL Parameters

- `orderId`: The ID of the order to reserve a fulfillment amount for

#### Request Body

```json
{
  "amount": 149
}
```

#### Response

```json
{
  "success": true,
  "data": {
    "orderId": "6123456789abcdef12345678",
    "reference": "ORD-123456-ABCDEF",
    "reservedAmount": 149,
    "expiresAt": "2023-01-01T00:05:00.000Z",
    "devProductId": "12345678",
    "gameId": "1234567890",
    "gameName": "Example Game",
    "reservationTimeMinutes": 5,
    "fulfillment": {
      "username": "ExampleUser",
      "userId": "123456789"
    }
  },
  "message": "Successfully reserved 149 Robux for fulfillment until 2023-01-01T00:05:00.000Z"
}
```

### Cancel Reservation

```
DELETE /api/merchant/orders/:orderId/reserve
```

Cancel an existing reservation if you can't complete the fulfillment in time.

#### URL Parameters

- `orderId`: The ID of the order for which to cancel the reservation

#### Response

```json
{
  "success": true,
  "message": "Reservation canceled successfully"
}
```

### Report Fulfillment

```
POST /api/merchant/orders/:orderId/fulfill
```

Report a fulfillment transaction for an order. This indicates that you have delivered a specific amount of Robux to the user. You must have a valid reservation before reporting fulfillment.

#### URL Parameters

- `orderId`: The ID of the order to fulfill

#### Request Body

```json
{
  "fulfillerId": "123456789", // Roblox ID of the account that delivered the Robux
  "fulfillerUsername": "FulfillmentBot", // Roblox username of the account that delivered the Robux
  "gameId": "1234567890", // ID of the game where the purchase was made
  "gameName": "Example Game", // Name of the game where the purchase was made
  "devProductId": "dp_123456", // ID of the developer product that was purchased
  "amount": 149, // Amount of Robux that was delivered
  "expectedAmount": 149, // Expected amount based on the developer product value
  "checkpointId": "checkout-1", // Optional: ID of the checkout location
  "devProductName": "149 Robux", // Optional: Name of the developer product
  "notes": "Delivered via API" // Optional: Notes about this fulfillment
}
```

#### Response

```json
{
  "success": true,
  "data": {
    "orderId": "6123456789abcdef12345678",
    "reference": "ORD-123456-ABCDEF",
    "transactionId": "6123456789abcdef87654321",
    "amount": 149,
    "fulfillmentStatus": "partial",
    "amountFulfilled": 600,
    "amountRemaining": 400,
    "fulfillmentPercentage": 60,
    "isCompleted": false
  }
}
```

### Get Order Status

```
GET /api/merchant/orders/:orderId
```

Get the current status of an order assigned to your merchant account.

#### URL Parameters

- `orderId`: The ID of the order to check

#### Response

```json
{
  "success": true,
  "data": {
    "id": "6123456789abcdef12345678",
    "reference": "ORD-123456-ABCDEF",
    "username": "ExampleUser",
    "gameId": "1234567890",
    "gameName": "Example Game",
    "robuxAmount": 1000,
    "amountFulfilled": 600,
    "amountRemaining": 400,
    "fulfillmentPercentage": 60,
    "status": "paid",
    "fulfillmentStatus": "partial",
    "createdAt": "2023-01-01T00:00:00.000Z",
    "updatedAt": "2023-01-01T01:00:00.000Z"
  }
}
```

### Release Order

```
POST /api/merchant/orders/:orderId/release
```

Release an order that you previously obtained through the fulfillment API. This allows other merchants to fulfill the remaining amount.

#### URL Parameters

- `orderId`: The ID of the order to release

#### Response

```json
{
  "success": true,
  "message": "Order released successfully"
}
```

## Error Responses

All API endpoints return standardized error responses:

```json
{
  "success": false,
  "message": "Error message explaining what went wrong"
}
```

Common HTTP status codes:

- `400`: Bad request (invalid parameters)
- `401`: Unauthorized (invalid API key)
- `403`: Forbidden (insufficient permissions)
- `404`: Not found (order not found)
- `409`: Conflict (reservation already exists)
- `500`: Internal server error

## Reservation System

The API uses a reservation system to prevent race conditions when multiple merchants try to fulfill the same order:

1. Find available orders using `GET /api/merchant/orders/fulfillment?robux_amount=149`
2. Reserve a specific amount using `POST /api/merchant/orders/:orderId/reserve`
3. You have 5 minutes to complete the fulfillment before the reservation expires
4. Report the fulfillment using `POST /api/merchant/orders/:orderId/fulfill`
5. If you can't complete the fulfillment in time, cancel the reservation

Reservations are automatically removed after successful fulfillment or after they expire.

## Order States

Orders go through several states during the fulfillment process:

1. `pending`: Order is created but not paid
2. `paid`: Order is paid and ready for fulfillment
3. `processing`: Fulfillment is in progress
4. `partial`: Partially fulfilled (some Robux delivered)
5. `completed`: Fully fulfilled (all Robux delivered)
6. `failed`: Payment or fulfillment failed

## Fulfillment Workflow

Here's a typical fulfillment workflow:

1. Find an order to fulfill using `GET /api/merchant/orders/fulfillment?robux_amount=149`
2. Reserve the amount using `POST /api/merchant/orders/:orderId/reserve` with `{"amount": 149}`
3. Deliver the Robux to the user (outside the API)
4. Report the fulfillment using `POST /api/merchant/orders/:orderId/fulfill`
5. If more Robux remain to be delivered, repeat the process or release the order

## Rate Limits

To ensure fair usage and system stability, API endpoints have rate limits:

- 60 requests per minute
- 1,000 requests per hour

Exceeding these limits will result in a `429 Too Many Requests` response.
