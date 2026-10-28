# Shop / Mall (`shop`)

> Full ecommerce core: products, categories, SKU / specs, cart, checkout,
> orders, shipping, pricing rules, and AI product description.

- **Identifier:** `shop`
- **Version:** 2.2.1
- **Category:** shop
- **Menu group:** Business Center → Shop
- **Min app version:** 0.10.0
- **Compatible editions:** enterprise, standard, pro, finance, minipro
- **Agent role:** `business`
- **Site capability:** `ecommerce`
- **Python deps:** none

---

## Core Features

### 1. Product management
- `routes/admin.py` (76 KB) — admin.
- `products` table.
- `product_specs` / `product_spec_values` / `product_skus`.
- AI description: `shop_ai_provider=deepseek`, `shop_ai_model=deepseek-chat`.

### 2. Categories
- `categories` — tree.

### 3. Cart
- `carts` table.

### 4. Orders
- `order_items` / `order_shipping`.
- `user_purchases` — for reviews / coupons.

### 5. Pricing rules
- `pricing_rules` table.

### 6. Shipping / express companies
- `express_companies` table.
- `order_shipping` binds waybill.

### 7. Rate limiting
- `rate_limits` table.

### 8. Public storefront
- `routes/public.py` (54 KB) — list / detail / cart / checkout / orders.
- Templates: `shop_detail.html` (19.7 KB), `orders.html` (23 KB).

### 9. Hooks
- Provided: `shop/products`, `shop/orders`, `shop/categories`.

---

## At a glance

| Area | Value |
|---|---|
| Storage | PG schema `shop`, **12 tables** |
| Blueprints | `/admin/shop` (`routes/admin.py`, 76 KB) + `/shop` (`routes/public.py`, 54 KB) |
| Admin UI | shop_products.html (20.8 KB), shop_orders.html (16.5 KB), shop_categories.html (36.2 KB) |
| Public templates | shop_detail.html (19.7 KB), orders.html (23 KB) |
| Python deps | none |
| Permissions | shop.manage |
| Capabilities | shop.product.manage, shop.order.manage, shop.category.manage, shop.checkout |
| Site capability | ecommerce |
| Hooks provided | shop/products, shop/orders, shop/categories |
| Dashboard | total_products / total_orders / orders_today / revenue_today |

### 12 tables

| Table | Purpose |
|---|---|
| `products` | Product master |
| `categories` | Category tree |
| `carts` | Cart |
| `user_purchases` | Purchase records |
| `order_items` | Order line items |
| `order_shipping` | Order shipping |
| `product_specs` | Spec names |
| `product_spec_values` | Spec values |
| `product_skus` | SKUs |
| `pricing_rules` | Pricing rules |
| `express_companies` | Express companies |
| `rate_limits` | Rate limits |

### Config (2 items)

| Key | Default | Purpose |
|---|---|---|
| `shop_ai_provider` | deepseek | AI provider (deepseek/openai/qwen etc.) |
| `shop_ai_model` | deepseek-chat | AI model name |

---

## Five-minute quick start

1. Enable — auto-creates 12 tables.
2. Open **Business Center → Shop**.
3. Add category → add products / SKUs.
4. Configure shipping / pricing rules.
5. Storefront browse / cart / checkout.

---

## Documents

| Document | Audience | Location |
|---|---|---|
| This file | E-commerce ops | `README.md` |
| Technical whitepaper | Developers | [`docs/whitepaper.md`](docs/whitepaper.md) |
| User guide / operation manual | Ops | [`docs/user-guide.md`](docs/user-guide.md) |
| Changelog | All | [`CHANGELOG.md`](CHANGELOG.md) |
