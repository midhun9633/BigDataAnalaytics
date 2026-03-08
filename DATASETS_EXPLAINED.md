# Understanding the Datasets – Olist Brazilian E-Commerce

> A plain-language breakdown of all 8 datasets used in the pipeline.

---

## Table of Contents

1. [orders – The Master Record](#1-orders--the-master-record)
2. [order_items – What Was Inside Each Order](#2-order_items--what-was-inside-each-order)
3. [payments – How the Order Was Paid](#3-payments--how-the-order-was-paid)
4. [customers – Who the Buyers Are](#4-customers--who-the-buyers-are)
5. [products – The Product Catalogue](#5-products--the-product-catalogue)
6. [reviews – Customer Feedback](#6-reviews--customer-feedback)
7. [sellers – The Marketplace Vendors](#7-sellers--the-marketplace-vendors)
8. [cat_trans – Category Name Translation](#8-cat_trans--category-name-translation)
9. [How They All Connect](#how-they-all-connect)
10. [Quick Mental Model](#quick-mental-model)

---

## The Big Picture

Think of Olist as a **marketplace** (like Amazon Brazil). These 8 tables are how their database stores everything that happens when a customer places an order.

---

## 1. `orders` — The Master Record
**99,441 rows | 8 columns**

This is the **central table**. Every order gets one row here. Everything else links back to it.

| Column | What it means | Example |
|---|---|---|
| `order_id` | Unique ID for the order (32-char hex) | `e481f51cbdc54678...` |
| `customer_id` | Who placed it | `9ef432eb6251297a...` |
| `order_status` | Current state | `delivered`, `cancelled`, `shipped` |
| `order_purchase_timestamp` | When the customer clicked "Buy" | `2017-10-02 10:56:33` |
| `order_approved_at` | When payment cleared | `2017-10-02 11:07:15` |
| `order_delivered_carrier_date` | When handed to courier | `2017-10-04 19:55:00` |
| `order_delivered_customer_date` | When customer received it | `2017-10-10 21:25:13` |
| `order_estimated_delivery_date` | Promised delivery date | `2017-10-18 00:00:00` |

> **Key insight:** The gap between `order_delivered_customer_date` and `order_estimated_delivery_date` is how we calculate if an order was **late or early**.

---

## 2. `order_items` — What Was Inside Each Order
**112,650 rows | 7 columns**

One row **per item** in an order. An order can have multiple items (hence 112k rows vs 99k orders).

| Column | What it means | Example |
|---|---|---|
| `order_id` | Links to orders table | same hex ID |
| `order_item_id` | Item number within the order (1, 2, 3...) | `1` |
| `product_id` | Which product | links to products |
| `seller_id` | Which seller sold it | links to sellers |
| `shipping_limit_date` | Seller's deadline to ship | `2017-10-06 11:07:15` |
| `price` | Product price (BRL) | `R$ 58.90` |
| `freight_value` | Shipping cost (BRL) | `R$ 13.29` |

> **Key insight:** `price + freight_value` = what the customer actually paid per item. This is how we calculate **basket value**.

---

## 3. `payments` — How the Order Was Paid
**103,886 rows | 5 columns**

One row per **payment method used**. If someone paid R$200 in two credit card installments, that's **2 rows** for the same order.

| Column | What it means | Example |
|---|---|---|
| `order_id` | Links to orders | same hex ID |
| `payment_sequential` | Which payment attempt (1, 2, 3...) | `1` |
| `payment_type` | Method used | `credit_card`, `boleto`, `voucher`, `debit_card` |
| `payment_installments` | Number of installments | `3` (paid in 3 months) |
| `payment_value` | Amount for this row (BRL) | `R$ 107.78` |

> **Key insight:** You must **SUM** `payment_value` per `order_id` to get the total order value. 3,039 orders have multiple payment rows — joining naively would double-count.

---

## 4. `customers` — Who the Buyers Are
**99,441 rows | 5 columns**

One row per customer account. Olist uses **two different IDs** — this is important.

| Column | What it means | Example |
|---|---|---|
| `customer_id` | ID used to link to `orders` (changes per order) | `9ef432eb...` |
| `customer_unique_id` | The actual person's permanent ID | `861eff4f...` |
| `customer_zip_code_prefix` | First 5 digits of postcode | `14409` |
| `customer_city` | City | `franca` |
| `customer_state` | State abbreviation | `SP` (São Paulo) |

> **Key insight:** `customer_id` is **not** the person — it's the session/order. The same person ordering twice gets two `customer_id` values. `customer_unique_id` is the real person. This is why purchase frequency analysis requires care.

---

## 5. `products` — The Product Catalogue
**32,951 rows | 9 columns**

One row per product. Notice: **no price here** — price is in `order_items` because the same product could sell at different prices over time.

| Column | What it means | Example |
|---|---|---|
| `product_id` | Unique product ID | `1e9e8ef04dbcff45...` |
| `product_category_name` | Category in Portuguese | `perfumaria` |
| `product_name_lenght` | Character count of the name | `40` |
| `product_description_lenght` | Character count of description | `287` |
| `product_photos_qty` | How many photos the listing has | `1` |
| `product_weight_g` | Weight in grams | `700` |
| `product_length_cm` | Length in cm | `25` |
| `product_height_cm` | Height in cm | `13` |
| `product_width_cm` | Width in cm | `20` |

> **Key insight:** Categories are in Portuguese — the `cat_trans` table translates them to English.

---

## 6. `reviews` — Customer Feedback
**99,224 rows | 7 columns**

One row per review. Not every order gets a review (hence 99,224 vs 99,441 orders).

| Column | What it means | Example |
|---|---|---|
| `review_id` | Unique review ID | `7bc2406110b926393...` |
| `order_id` | Links to orders | same hex ID |
| `review_score` | Star rating | `1` to `5` |
| `review_comment_title` | Short title (often blank) | `"Recommend!"` |
| `review_comment_message` | Full review text (often blank) | `"Arrived on time"` |
| `review_creation_date` | When the review form was sent | `2018-01-18 00:00:00` |
| `review_answer_timestamp` | When the customer submitted it | `2018-01-22 21:46:24` |

> **Key insight:** Most customers don't leave text — only a score. Null `review_comment_message` means no text was written, not a data error.

---

## 7. `sellers` — The Marketplace Vendors
**3,095 rows | 4 columns**

One row per seller registered on Olist. Small table — only 3,095 sellers serve 99k+ orders.

| Column | What it means | Example |
|---|---|---|
| `seller_id` | Unique seller ID | `3442f8959a84dda7...` |
| `seller_zip_code_prefix` | Seller's postcode (first 5) | `13023` |
| `seller_city` | Seller's city | `campinas` |
| `seller_state` | Seller's state | `SP` |

> **Key insight:** One seller can have hundreds of orders. We link this to `order_items` via `seller_id` to calculate **revenue per seller**.

---

## 8. `cat_trans` — Category Name Translation
**71 rows | 2 columns**

A simple lookup table. 71 Portuguese category names mapped to English.

| `product_category_name` (Portuguese) | `product_category_name_english` |
|---|---|
| `perfumaria` | `perfumery` |
| `esporte_lazer` | `sports_leisure` |
| `bebes` | `baby` |
| `utilidades_domesticas` | `housewares` |
| `instrumentos_musicais` | `musical_instruments` |
| `moveis_decoracao` | `furniture_decor` |

> **Key insight:** This is joined to `products` so all category labels in the analysis are readable in English.

---

## How They All Connect

```
customers ──────── orders ─────────── order_items ── products
                     │                     │               │
                  payments              sellers        cat_trans
                     │
                  reviews
```

| Join | Key | Type |
|---|---|---|
| `order_items` → `orders` | `order_id` | Many items per order |
| `orders` → `customers` | `customer_id` | Many orders per customer |
| `order_items` → `products` | `product_id` | Many items per product |
| `order_items` → `sellers` | `seller_id` | Many items per seller |
| `payments` → `orders` | `order_id` | Many payments per order |
| `reviews` → `orders` | `order_id` | One review per order |
| `products` → `cat_trans` | `product_category_name` | Translation lookup |

---

## Quick Mental Model

> A **customer** places an **order** → the order contains **items** (each from a **seller**, of a **product**) → the order is paid via **payments** → later the customer leaves a **review**. Product categories are translated via **cat_trans**.

### Common Pitfalls to Avoid

| Mistake | Why it breaks | Fix |
|---|---|---|
| Joining `payments` directly without summing | Double-counts installment rows | `groupby('order_id').sum()` first |
| Using `customer_id` for loyalty analysis | Same person = multiple IDs | Use `customer_unique_id` |
| Using `products.price` (it doesn't exist) | Price lives in `order_items` | Join `order_items` for price |
| Treating null review text as missing data | Customer just didn't write anything | Fill with empty string `''` |
| Leaving category names in Portuguese | Unreadable in analysis output | Join `cat_trans` for English names |
