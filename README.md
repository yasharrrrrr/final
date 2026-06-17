# BAORCO Online Shop — REST API

Backend for the **BAORCO** online furniture shop, built with **Django REST
Framework**. It powers a separate **admin panel** and an **Android customer
app** through one JSON API secured with **JWT**, includes an internal
**accounting system with installments**, **SMS OTP authentication** via
[sms.ir](https://sms.ir), and runs on **Microsoft SQL Server** (with a SQLite
fallback for development).

---

## Features

- **Phone-based authentication** — register / login with phone + password **or**
  one-time password (OTP) over SMS. JWT access & refresh tokens for the mobile app.
- **Strict admin / customer separation**
  - `/api/admin/…` — staff only, full control of the system.
  - `/api/shop/…` — customer storefront, cart, checkout, own orders & invoices.
  - The admin **grants or revokes** any customer's access (`activate` /
    `deactivate`), resets passwords and changes roles.
- **Catalog** — nested categories, furniture products with dimensions/material,
  images, stock control.
- **Orders** — server-side cart, atomic checkout with stock locking, order status workflow.
- **Accounting** — invoices, **installment plans (خرید اقساطی)**, payment recording,
  and a simple double-entry **ledger** with a trial-balance summary.
- **SMS gateway (sms.ir)** — OTP template `769673` and order-report template `800879`.

---

## Quick start (development — SQLite)

```bash
# 1. create & activate a virtualenv
python -m venv .venv
source .venv/Scripts/activate      # Windows Git Bash
# .venv\Scripts\activate           # Windows PowerShell/cmd

# 2. install dependencies
pip install -r requirements.txt

# 3. configure environment
cp .env.example .env               # keep DB_ENGINE=sqlite for local dev

# 4. migrate & seed
python manage.py makemigrations
python manage.py migrate
python manage.py seed_demo         # demo admin + customer + products

# 5. run
python manage.py runserver 0.0.0.0:8000
```

Demo accounts created by `seed_demo`:

| Role     | Phone        | Password       |
|----------|--------------|----------------|
| Admin    | 09120000000  | `Admin@1234`   |
| Customer | 09121111111  | `Customer@1234`|

Django admin site: <http://localhost:8000/admin/>

---

## Switching to Microsoft SQL Server

1. Install the **ODBC Driver 17 (or 18) for SQL Server** on the host.
2. In `.env` set:
   ```env
   DB_ENGINE=mssql
   DB_NAME=baorco
   DB_USER=sa
   DB_PASSWORD=YourStrong!Passw0rd
   DB_HOST=localhost
   DB_PORT=1433
   DB_DRIVER=ODBC Driver 17 for SQL Server
   ```
3. `python manage.py migrate`

The driver (`mssql-django`) is already in `requirements.txt`.

---

## SMS (sms.ir) configuration

Set in `.env` (sandbox is on by default so test messages don't cost credit):

```env
SMSIR_API_KEY=...your live key...
SMSIR_SANDBOX_API_KEY=...your sandbox key...
SMSIR_USE_SANDBOX=True
SMSIR_OTP_TEMPLATE_ID=769673
SMSIR_ORDER_TEMPLATE_ID=800879
```

- **OTP template 769673** parameters: `TIME`, `TOKEN`
- **Order report template 800879** parameters: `TIME`, `QUNTITY`, `PRICE`, `YES`,
  `DVISION`, `TIME_SHAMSI` (the due date is rendered as a Jalali/Shamsi date)

Every send is logged in the `sms_smslog` table for auditing.

---

## API map

### Auth — `/api/auth/`
| Method | Path | Purpose |
|--------|------|---------|
| POST | `register/` | Register customer (phone+password), sends OTP |
| POST | `otp/request/` | Request an OTP (`purpose`: login/register/reset) |
| POST | `otp/verify/` | Verify OTP → returns JWT tokens |
| POST | `login/` | Phone + password → JWT pair |
| POST | `token/refresh/` | Refresh access token |
| GET/PATCH | `me/` | Current user |
| GET/PUT | `me/profile/` | Shipping/identity profile |
| POST | `password/change/` | Change own password |

### Admin — `/api/admin/` (staff only)
| Resource | Path |
|----------|------|
| Users | `accounts/users/` (+ `activate/`, `deactivate/`, `set_role/`, `reset_password/`) |
| Categories | `catalog/categories/` |
| Products | `catalog/products/`, `catalog/images/` |
| Orders | `orders/orders/` (+ `set_status/`) |
| Invoices | `accounting/invoices/` (+ `record_payment/`) |
| Payments | `accounting/payments/` |
| Ledger | `accounting/ledger/` (+ `summary/`) |

### Shop — `/api/shop/` (customer)
| Resource | Path |
|----------|------|
| Catalog | `catalog/categories/`, `catalog/products/` (read-only) |
| Cart | `orders/cart/`, `orders/cart/items/` |
| Checkout | `orders/checkout/` |
| My orders | `orders/orders/` |
| My invoices | `accounting/invoices/` |
| My installments | `accounting/installments/` |

All `/api/admin/` and `/api/shop/` endpoints require
`Authorization: Bearer <access_token>`.

---

## Android client notes

- The Android emulator reaches the host machine at `http://10.0.2.2:8000`.
- Store the JWT `access` token and send it as `Authorization: Bearer …`.
- Refresh via `POST /api/auth/token/refresh/` when you get a 401.
- Typical mobile flow: `otp/request` → `otp/verify` (get tokens) →
  browse `shop/catalog/products` → `cart/items` → `checkout`.

---

## Project layout

```
baorco/        project settings, root URL split (admin vs shop)
core/          shared permissions (IsAdmin/IsCustomer), pagination, base model
accounts/      custom phone User, OTP, JWT auth, admin user-management
catalog/       categories, products, images
orders/        cart, checkout, orders (admin + customer)
accounting/    invoices, installment plans, payments, ledger, Jalali dates
sms/           sms.ir gateway client + send log
```
