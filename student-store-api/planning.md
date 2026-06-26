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

## Decision Logs

### Product Model Decisions

- **Schema translation that went smoothly**: `id` + `@default(autoincrement())` and required strings mapped directly from spec to Prisma with no conversion issues.
- **Field decision I made during implementation that was not in the original spec**: Kept `price` as `Float` to match the current course scaffold and seed format, while validating `price` as numeric in create/update route handlers.
- **Route behavior that needed a spec update**: Added explicit query parameter behavior for `GET /products` (`category`, `sort`, and defaults), and documented how invalid `sort` and unmatched `category` values are handled.

## Spec Reconciliation -- Milestone 4 (Schema Audit)

### Schema vs. spec gaps found

- Initially, `schema.prisma` had `Product` and `Order` but was missing `OrderItem`; added `OrderItem` with all required fields (`orderId`, `productId`, `quantity`, `price`, timestamps).
- Added missing relation array fields in parent models (`Product.orderItems` and `Order.orderItems`) so schema now reflects the one-to-many relationships documented in the Data Models section.
- `OrderItem.price` remained `Float` and is explicitly treated as purchase-time unit price in route/model logic, consistent with the spec.

### Cascade delete verification

- Deleting a Product removes associated OrderItems: ✅ tested
- Deleting an Order removes associated OrderItems: ✅ tested

### Order Creation Transaction Decisions

- **What my Transactional Flow spec got right**: The sequence in the spec matched implementation: validate body, verify product IDs, compute total, create order with items, return order with `orderItems`.
- **What the spec missed that I discovered during implementation**: I needed explicit validation for each item (`productId` integer and `quantity` > 0), so malformed items return a clear `400` before any DB write.
- **How the transaction error handling works**: `prisma.$transaction` wraps all writes as one unit; if any step throws (like missing product), Prisma rolls back all writes so no partial order or orphaned order items are saved.
- **One thing I'd design differently if starting over**: I would add an explicit `ORDER_STATUS` enum in Prisma instead of free-form `String` for `status`, to prevent invalid status values at the schema level.

## Final Spec Reconciliation: Project Complete

### Full-system audit result

- Product browsing flow aligns with contract: frontend reads `GET /products` response as `{ products: [...] }` and item detail reads `GET /products/:productId` as `{ product: {...} }`.
- Checkout flow aligns with contract: frontend sends `POST /orders` body with `customer`, `status`, and `items`, and receives `{ order: {..., orderItems: [...] } }`.
- Added backend CORS middleware (`app.use(cors())`) to allow frontend integration from a separate dev origin.

### Gaps resolved during frontend integration

- Frontend scaffold originally expected a different checkout response (`order.purchase.receipt`) and no longer matched the API contract; UI now renders confirmation from the actual API order shape (`id`, `customer`, `status`, `totalPrice`, `orderItems`).
- `ProductDetail` previously referenced `image_url`; backend returns `imageUrl`, so the component was updated to use `imageUrl`.
- Payment form state mapping was inconsistent (`id`/`email` instead of `name`/`dorm_number`), which prevented clean request payload creation; corrected to match app state and checkout parsing.

### What the spec enabled during this project

The written API contract made it faster to identify where frontend assumptions diverged from backend responses, especially around checkout response shape and product field naming. Having the transaction flow documented up front also reduced debugging time by making rollback and error-case expectations explicit before testing end-to-end.

