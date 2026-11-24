# remy_api — Database diagram

Entity-relationship diagram of the SQLAlchemy models in `remy_api/src/models/`.
Snapshot taken 2026-11-18 from `origin/main` (`src/models` identical to the `diego3008/fix-assistant-id` worktree).
Re-check the models before relying on it: schema changes ship with an Alembic migration.

```mermaid
erDiagram
    BUSINESSES ||--o{ MENUS : "has (CASCADE)"
    BUSINESSES ||--o{ CUSTOMERS : "has (CASCADE)"
    BUSINESSES ||--o{ ORDERS : "receives (RESTRICT)"
    MENUS ||--o{ DISHES : "contains (CASCADE)"
    CUSTOMERS ||--o{ ORDERS : "places (RESTRICT)"
    ORDERS ||--|{ ORDER_ITEMS : "has (CASCADE)"
    DISHES ||--o{ ORDER_ITEMS : "ordered as (RESTRICT)"

    BUSINESSES {
        int id PK
        string name
        bool b_active
        string phone
        string timezone "default America/Monterrey"
        datetime created_at
        datetime updated_at
    }

    MENUS {
        int id PK
        int business_id FK
        string name "String(80), default Main"
        bool is_active
        datetime created_at
        datetime updated_at
    }

    DISHES {
        int id PK
        int menu_id FK
        string name "String(120)"
        text description "nullable"
        decimal price "Numeric(10,2), CHECK >= 0"
        bool is_available
        datetime created_at
        datetime updated_at
    }

    CUSTOMERS {
        int id PK
        int business_id FK "UNIQUE with phone"
        string phone "E.164, UNIQUE with business_id"
        string name "nullable"
        text default_address "nullable"
        datetime created_at
        datetime updated_at
    }

    ORDERS {
        int id PK
        int business_id FK
        int customer_id FK
        order_channel channel "telegram, whatsapp, phone"
        order_fulfillment fulfillment "delivery, pickup"
        order_status status "pending, confirmed, preparing, ready, completed, cancelled"
        text delivery_address "nullable, required if delivery"
        decimal total "Numeric(10,2)"
        string thread_id "nullable, LangGraph thread (no FK)"
        datetime created_at
        datetime updated_at
    }

    ORDER_ITEMS {
        int id PK
        int order_id FK
        int dish_id FK
        int quantity "CHECK > 0, default 1"
        decimal unit_price "price snapshot at order time"
        text notes "nullable"
    }

    CONVERSATIONS {
        int id PK
        uuid chat_id "no FK"
        uuid thread_id "no FK, LangGraph thread"
        uuid parent_thread_id "no FK"
        bool b_active
        int message_count
        datetime created_at
        datetime updated_at
    }
```

## Notes

- **`conversations` is an island.** It has no foreign keys. `chat_id` is typed as a UUID, even though Telegram chat IDs
  are ints. The agent flow doesn't use this table: the thread lookup goes through LangGraph metadata.
- **`orders.thread_id` is a soft link.** It is a plain string pointing at a LangGraph thread, not a foreign key.
- **Delete behaviour:** deleting a business, menu or order cascades down the tree. Deleting a customer or dish that
  orders still reference is blocked (RESTRICT, which the API returns as 409).
- **Constraints:** `ck_order_delivery_needs_address` (delivery orders need an address), `uq_customer_business_phone`,
  `ck_dish_price_nonnegative`, `ck_order_item_quantity_positive`, and the index `ix_orders_business_status`.
- **Timestamps:** `order_items` has none (no `TimestampMixin`). `businesses` and `conversations` have no server-side
  defaults, so the route sets them.
- **Not shown:** `incoming_message.py` holds Pydantic classes (`IncomingMessage`, `AgentResponse`), not tables.
