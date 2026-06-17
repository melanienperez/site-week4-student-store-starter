# Student Store API System Spec

This document defines the data model and API contract before implementation. It is the source of truth for schema and route decisions in this project.

## Section 1: Data Models

### Product

- **Primary key:** `id` (`Int`, required, auto-increment, `@id @default(autoincrement())`)
- **Fields:**
  - `id`: `Int`, required, default `autoincrement()`
  - `name`: `String`, required
  - `description`: `String`, required
  - `price`: `Float`, required
  - `imageUrl`: `String`, required
  - `category`: `String`, required
  - `createdAt`: `DateTime`, required, default `now()`
  - `updatedAt`: `DateTime`, required, auto-updated (`@updatedAt`)
- **Relationships:**
  - One `Product` has many `OrderItem` records (`orderItems OrderItem[]`)
- **Cascade behavior:**
  - If a `Product` is deleted, all `OrderItem` rows that reference that product are also deleted.
  - This is enforced on the `OrderItem.productId -> Product.id` relation with `onDelete: Cascade`.

### Order

- **Primary key:** `id` (`Int`, required, auto-increment, `@id @default(autoincrement())`)
- **Fields:**
  - `id`: `Int`, required, default `autoincrement()`
  - `customer`: `Int`, required
  - `totalPrice`: `Float`, required, default `0`
  - `status`: `String`, required (expected values: `pending`, `completed`, `cancelled`)
  - `createdAt`: `DateTime`, required, default `now()`
  - `updatedAt`: `DateTime`, required, auto-updated (`@updatedAt`)
- **Relationships:**
  - One `Order` has many `OrderItem` records (`orderItems OrderItem[]`)
- **Cascade behavior:**
  - If an `Order` is deleted, all `OrderItem` rows that reference that order are also deleted.
  - This is enforced on the `OrderItem.orderId -> Order.id` relation with `onDelete: Cascade`.

### OrderItem

- **Primary key:** `id` (`Int`, required, auto-increment, `@id @default(autoincrement())`)
- **Fields:**
  - `id`: `Int`, required, default `autoincrement()`
  - `orderId`: `Int`, required (foreign key -> `Order.id`)
  - `productId`: `Int`, required (foreign key -> `Product.id`)
  - `quantity`: `Int`, required
  - `price`: `Float`, required (unit price captured at purchase time)
  - `createdAt`: `DateTime`, required, default `now()`
  - `updatedAt`: `DateTime`, required, auto-updated (`@updatedAt`)
- **Relationships:**
  - Many `OrderItem` belong to one `Order`
  - Many `OrderItem` belong to one `Product`
- **Cascade behavior:**
  - Deleting parent `Order` cascades delete to child `OrderItem`.
  - Deleting parent `Product` cascades delete to child `OrderItem`.
  - `OrderItem` is the intersection model that depends on both parent tables.

## Section 2: API Contract

### Global error shape

All API errors use this shape:

```json
{ "error": "Human-readable error message" }
```

### Products endpoints

#### GET `/products`

- **Request shape:** no body; optional query params can be added later (for example, `category`)
- **Success response:**
  - `200 OK`
  - body:
    ```json
    {
      "products": [
        {
          "id": 1,
          "name": "College Hoodie",
          "description": "Comfortable and stylish hoodie with the college logo.",
          "price": 29.99,
          "imageUrl": "https://tinyurl.com/college-hoodie",
          "category": "Apparel"
        }
      ]
    }
    ```
- **Error case example:**
  - `500 Internal Server Error`
  - `{ "error": "Failed to fetch products" }`

#### GET `/products/:productId`

- **Request shape:** route param `productId` (`Int`)
- **Success response:**
  - `200 OK`
  - body:
    ```json
    {
      "product": {
        "id": 1,
        "name": "College Hoodie",
        "description": "Comfortable and stylish hoodie with the college logo.",
        "price": 29.99,
        "imageUrl": "https://tinyurl.com/college-hoodie",
        "category": "Apparel"
      }
    }
    ```
- **Error case example:**
  - `404 Not Found`
  - `{ "error": "Product not found" }`

### Orders endpoints

#### GET `/orders`

- **Request shape:** no body
- **Success response:**
  - `200 OK`
  - body includes order rows and their items:
    ```json
    {
      "orders": [
        {
          "id": 1,
          "customer": 101,
          "totalPrice": 89.97,
          "status": "completed",
          "createdAt": "2023-04-06T10:00:00.000Z",
          "orderItems": [
            { "id": 1, "productId": 1, "quantity": 2, "price": 29.99 }
          ]
        }
      ]
    }
    ```
- **Error case example:**
  - `500 Internal Server Error`
  - `{ "error": "Failed to fetch orders" }`

#### GET `/orders/:orderId`

- **Request shape:** route param `orderId` (`Int`)
- **Success response:**
  - `200 OK`
  - body:
    ```json
    {
      "order": {
        "id": 1,
        "customer": 101,
        "totalPrice": 89.97,
        "status": "completed",
        "orderItems": [
          { "id": 1, "productId": 1, "quantity": 2, "price": 29.99 }
        ]
      }
    }
    ```
- **Error case example:**
  - `404 Not Found`
  - `{ "error": "Order not found" }`

#### POST `/orders`

- **Request shape (body):**
  ```json
  {
    "customer": 101,
    "status": "pending",
    "items": [
      { "productId": 1, "quantity": 2 },
      { "productId": 4, "quantity": 1 }
    ]
  }
  ```
- **Success response:**
  - `201 Created`
  - body returns created order with order items:
    ```json
    {
      "order": {
        "id": 3,
        "customer": 101,
        "status": "pending",
        "totalPrice": 61.97,
        "orderItems": [
          { "id": 8, "orderId": 3, "productId": 1, "quantity": 2, "price": 29.99 },
          { "id": 9, "orderId": 3, "productId": 4, "quantity": 1, "price": 1.99 }
        ]
      }
    }
    ```
- **Error case example:**
  - `400 Bad Request` for malformed body
  - `{ "error": "items must be a non-empty array" }`
  - `404 Not Found` if one or more `productId` values do not exist
  - `{ "error": "One or more products not found" }`

#### PUT `/orders/:orderId`

- **Request shape:** route param `orderId`; body fields to update (at minimum status)
  ```json
  { "status": "completed" }
  ```
- **Success response:**
  - `200 OK`
  - body:
    ```json
    { "order": { "id": 3, "status": "completed" } }
    ```
- **Error case example:**
  - `404 Not Found`
  - `{ "error": "Order not found" }`

#### DELETE `/orders/:orderId`

- **Request shape:** route param `orderId`
- **Success response:**
  - `200 OK`
  - body:
    ```json
    { "deleted": true }
    ```
- **Error case example:**
  - `404 Not Found`
  - `{ "error": "Order not found" }`

### OrderItems endpoint (read-only helper)

#### GET `/order-items`

- **Request shape:** no body; optional query params `orderId` or `productId`
- **Success response:**
  - `200 OK`
  - body:
    ```json
    {
      "orderItems": [
        { "id": 1, "orderId": 1, "productId": 1, "quantity": 2, "price": 29.99 }
      ]
    }
    ```
- **Error case example:**
  - `500 Internal Server Error`
  - `{ "error": "Failed to fetch order items" }`

## Section 3: Transactional Flow (POST `/orders`)

### Goal

Create an order and all of its order items atomically, with a computed `totalPrice`.

### Request body

```json
{
  "customer": 101,
  "status": "pending",
  "items": [
    { "productId": 1, "quantity": 2 },
    { "productId": 4, "quantity": 1 }
  ]
}
```

### Step-by-step data-layer flow

1. Validate request body:
   - `customer` exists and is numeric
   - `status` is valid
   - `items` is a non-empty array
   - each item has numeric `productId` and positive integer `quantity`
2. Start a database transaction with Prisma (`prisma.$transaction`).
3. Query all products referenced by `items` to ensure every `productId` exists.
4. If any product is missing, throw an error inside the transaction and abort.
5. Build a map of `productId -> current product price`.
6. Calculate `totalPrice`:
   - sum of `(item.quantity * product.price)` across all items
7. Create `Order` first with `customer`, `status`, and computed `totalPrice`.
8. Create `OrderItem` rows linked to the new `Order.id`:
   - each row stores `orderId`, `productId`, `quantity`, and unit `price` at checkout time
9. Return the created `Order` including `orderItems` (and optionally related product data).
10. Commit transaction only after all operations succeed.

### Failure behavior (atomicity)

- If creating any `OrderItem` fails, or if any `productId` does not exist, Prisma rolls back the entire transaction.
- Result: no partial order, no partial items.
- API response for nonexistent product reference:
  - `404 Not Found`
  - `{ "error": "One or more products not found" }`

### Notes

- `totalPrice` is stored on `Order` as a snapshot value for historical reporting.
- `OrderItem.price` stores purchase-time price so future product price changes do not rewrite historical orders.
