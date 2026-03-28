# Pricing Structure - uSipipo Ecosystem

> Comprehensive documentation of all pricing structures, conversion rates, and formulas for the uSipipo VPN ecosystem.

**Last Updated:** March 27, 2026  
**Version:** 1.0.0

---

## Table of Contents

1. [Exchange Rate](#exchange-rate)
2. [Data Packages](#data-packages)
3. [Data Slots](#data-slots)
4. [Subscription Plans](#subscription-plans)
5. [Consumption Billing](#consumption-billing)
6. [Payment Methods](#payment-methods)
7. [Calculations & Formulas](#calculations--formulas)

---

## Exchange Rate

### Base Conversion

| Constant | Value | Description |
|----------|-------|-------------|
| `STARS_PER_USDT` | **120** | 1 USDT = 120 Telegram Stars |

### Conversion Formula

```
USDT = Stars / 120
Stars = USDT × 120
```

### Examples

| Stars | USDT Equivalent |
|-------|-----------------|
| 120 ⭐ | $1.00 |
| 360 ⭐ | $3.00 |
| 600 ⭐ | $5.00 |
| 1200 ⭐ | $10.00 |
| 1800 ⭐ | $15.00 |

---

## Data Packages

Prepaid data packages purchasable with Telegram Stars or USDT. All packages include bonus data and 35-day validity.

### Package Comparison Table

| Package | Base Data | Bonus | Total Data | Stars | USDT | Price/GB | Discount vs Consumption |
|---------|-----------|-------|------------|-------|------|----------|------------------------|
| **Básico** | 10 GB | 0% | 10 GB | 250 ⭐ | $2.08 | $0.208/GB | 58% |
| **Estándar** | 30 GB | 10% | 33 GB | 600 ⭐ | $5.00 | $0.167/GB | 67% |
| **Avanzado** | 60 GB | 15% | 69 GB | 960 ⭐ | $8.00 | $0.133/GB | 73% |
| **Premium** | 120 GB | 20% | 144 GB | 1440 ⭐ | $12.00 | $0.100/GB | 80% |
| **Ilimitado** | 200 GB | 25% | 250 GB | 1800 ⭐ | $15.00 | $0.075/GB | 85% |

### Package Details

#### Básico
- **Package Type:** `BASIC`
- **Base Data:** 10 GB
- **Bonus:** 0% (no bonus)
- **Total Data:** 10 GB
- **Price:** 250 Stars / $2.08 USDT
- **Price per GB:** $0.208
- **Validity:** 35 days
- **Best for:** Light users, trial users

#### Estándar
- **Package Type:** `ESTANDAR`
- **Base Data:** 30 GB
- **Bonus:** 10% (+3 GB free)
- **Total Data:** 33 GB
- **Price:** 600 Stars / $5.00 USDT
- **Price per GB:** $0.167
- **Validity:** 35 days
- **Best for:** Regular users, best value entry-level

#### Avanzado
- **Package Type:** `AVANZADO`
- **Base Data:** 60 GB
- **Bonus:** 15% (+9 GB free)
- **Total Data:** 69 GB
- **Price:** 960 Stars / $8.00 USDT
- **Price per GB:** $0.133
- **Validity:** 35 days
- **Best for:** Heavy users, streamers

#### Premium
- **Package Type:** `PREMIUM`
- **Base Data:** 120 GB
- **Bonus:** 20% (+24 GB free)
- **Total Data:** 144 GB
- **Price:** 1440 Stars / $12.00 USDT
- **Price per GB:** $0.100
- **Validity:** 35 days
- **Best for:** Power users, content creators

#### Ilimitado
- **Package Type:** `UNLIMITED`
- **Base Data:** 200 GB
- **Bonus:** 25% (+50 GB free)
- **Total Data:** 250 GB
- **Price:** 1800 Stars / $15.00 USDT
- **Price per GB:** $0.075
- **Validity:** 35 days
- **Best for:** Extreme users, families, resellers

### Savings Analysis

| Package | vs Consumption Billing (at $0.50/GB) |
|---------|--------------------------------------|
| Básico | Save $2.92 (58% cheaper) |
| Estándar | Save $11.50 (67% cheaper) |
| Avanzado | Save $26.50 (73% cheaper) |
| Premium | Save $60.00 (80% cheaper) |
| Ilimitado | Save $110.00 (85% cheaper) |

---

## Data Slots

Additional VPN key slots for users who need multiple device connections.

### Slot Options Table

| Slot Package | Additional Keys | Stars | USDT | Price per Key |
|--------------|-----------------|-------|------|---------------|
| **+1 Clave** | +1 key | 300 ⭐ | $2.50 | $2.50/key |
| **+3 Claves** | +3 keys | 700 ⭐ | $5.83 | $1.94/key |
| **+5 Claves** | +5 keys | 1000 ⭐ | $8.33 | $1.67/key |

### Slot Details

#### +1 Clave
- **Additional Keys:** 1
- **Price:** 300 Stars / $2.50 USDT
- **Price per Key:** $2.50
- **Best for:** Single additional device

#### +3 Claves
- **Additional Keys:** 3
- **Price:** 700 Stars / $5.83 USDT
- **Price per Key:** $1.94 (22% savings)
- **Best for:** Small families, multiple devices

#### +5 Claves
- **Additional Keys:** 5
- **Price:** 1000 Stars / $8.33 USDT
- **Price per Key:** $1.67 (33% savings)
- **Best for:** Large families, teams, resellers

### Default Limits

| User Type | Max Keys |
|-----------|----------|
| Free User | 2 keys |
| Standard User | 10 keys (base) |
| With +5 Slot | 15 keys total |

---

## Subscription Plans

Monthly subscription plans with unlimited data and premium features.

### Subscription Comparison Table

| Plan | Duration | Stars | USDT | Price/Month | Bonus | Effective Monthly |
|------|----------|-------|------|-------------|-------|-------------------|
| **1 Month** | 30 days | 360 ⭐ | $2.99 | $2.99 | 0% | $2.99/month |
| **3 Months** | 90 days | 960 ⭐ | $7.99 | $2.66 | 11% | $2.39/month |
| **6 Months** | 180 days | 1560 ⭐ | $12.99 | $2.17 | 13% | $1.89/month |
| **12 Months** | 365 days | 2160 ⭐ | $24.99 | $2.08 | 16% | $1.75/month |

> **Note:** 12-month plan pricing based on Telegram bot fallback configuration.

### Plan Details

#### 1 Month
- **Plan Type:** `ONE_MONTH`
- **Duration:** 30 days (1 month)
- **Price:** 360 Stars / $2.99 USDT
- **Monthly Equivalent:** $2.99/month
- **Data Limit:** Unlimited
- **Features:**
  - 1 device
  - Standard speed
  - Basic support
- **Best for:** Short-term users, testing

#### 3 Months
- **Plan Type:** `THREE_MONTHS`
- **Duration:** 90 days (3 months)
- **Price:** 960 Stars / $7.99 USDT
- **Monthly Equivalent:** $2.66/month (11% discount)
- **Effective Monthly:** $2.39/month with bonus
- **Data Limit:** Unlimited
- **Features:**
  - 1 device
  - Standard speed
  - Basic support
- **Best for:** Quarterly users, travelers

#### 6 Months
- **Plan Type:** `SIX_MONTHS`
- **Duration:** 180 days (6 months)
- **Price:** 1560 Stars / $12.99 USDT
- **Monthly Equivalent:** $2.17/month (27% discount vs monthly)
- **Effective Monthly:** $1.89/month with bonus
- **Data Limit:** Unlimited
- **Features:**
  - 1 device
  - Standard speed
  - Basic support
- **Best for:** Long-term users, best value

#### 12 Months
- **Plan Type:** `TWELVE_MONTHS`
- **Duration:** 365 days (12 months)
- **Price:** 2160 Stars / $24.99 USDT
- **Monthly Equivalent:** $2.08/month (31% discount vs monthly)
- **Effective Monthly:** $1.75/month with bonus
- **Data Limit:** Unlimited
- **Features:**
  - 1 device
  - Standard speed
  - Basic support
- **Best for:** Annual subscribers, maximum savings

### Subscription Savings

| Plan | vs Monthly Pricing | Total Savings |
|------|-------------------|---------------|
| 1 Month | Baseline | $0.00 |
| 3 Months | 11% discount | $0.98 saved |
| 6 Months | 27% discount | $4.95 saved |
| 12 Months | 31% discount | $10.89 saved |

---

## Consumption Billing

Postpaid billing model where users pay only for what they consume.

### Pricing Structure

| Metric | Value |
|--------|-------|
| **Price per GB** | $0.50 USD |
| **Price per MB** | $0.0005 USD ($0.50 / 1024) |
| **Billing Cycle** | 30 days |
| **Minimum Charge** | $0.01 (1 MB) |
| **Maximum Charge** | Unlimited |

### Consumption Examples

| Data Consumed | Cost (USD) | Cost (Stars) |
|---------------|------------|--------------|
| 100 MB | $0.05 | 6 ⭐ |
| 500 MB | $0.25 | 30 ⭐ |
| 1 GB | $0.50 | 60 ⭐ |
| 5 GB | $2.50 | 300 ⭐ |
| 10 GB | $5.00 | 600 ⭐ |
| 20 GB | $10.00 | 1200 ⭐ |
| 50 GB | $25.00 | 3000 ⭐ |
| 100 GB | $50.00 | 6000 ⭐ |

### Billing Cycle Details

- **Cycle Duration:** 30 days from activation
- **Billing Type:** Postpaid (pay after consumption)
- **Payment Deadline:** End of billing cycle
- **Late Payment:** All VPN keys blocked until payment
- **Automatic Deactivation:** Mode turns off after payment

### Comparison with Packages

| Scenario | Consumption Billing | Package (Premium) | Savings |
|----------|---------------------|-------------------|---------|
| 5 GB/month | $2.50 | $12.00 (144 GB) | Package better |
| 10 GB/month | $5.00 | $12.00 (144 GB) | Package better |
| 20 GB/month | $10.00 | $12.00 (144 GB) | Package better |
| 50 GB/month | $25.00 | $12.00 (144 GB) | Package better |
| 100 GB/month | $50.00 | $12.00 (144 GB) | Package better |
| 150 GB/month | $75.00 | $12.00 (144 GB) | Package better |
| 200+ GB/month | $100.00+ | $15.00 (250 GB) | Package better |

### When to Use Consumption Billing

**Recommended for:**
- Users who consume < 2 GB/month
- Users who want unlimited data flexibility
- Users who trust themselves to monitor usage
- Temporary high-usage months

**Not Recommended for:**
- Regular heavy users (> 10 GB/month)
- Budget-conscious users
- Users who prefer predictable costs

---

## Payment Methods

### Supported Payment Methods

| Method | Currency | Processing Time | Minimum | Maximum | Fees |
|--------|----------|-----------------|---------|---------|------|
| **Telegram Stars** | Stars (XTR) | Instant | 1 ⭐ | 10,000 ⭐ | 0% |
| **Crypto (USDT)** | USDT (TRC20) | 5-30 minutes | $1.00 | $10,000 | Network fee |

### Telegram Stars

**Overview:**
- Native Telegram currency for digital goods
- Instant activation
- No transaction fees from uSipipo
- Purchased via Telegram's in-app purchase system

**Process:**
1. Select Stars payment option
2. Telegram invoice sent to user
3. User pays via Telegram (credit card, Apple Pay, Google Pay, etc.)
4. Service activated instantly upon payment confirmation

**Best for:**
- Small purchases (< $20)
- Instant activation needs
- Users without crypto wallets

### Crypto (USDT via TronDealer)

**Overview:**
- USDT on TRC20 (Tron) network
- Lower network fees than ERC20
- Confirmed in 5-30 minutes
- Ideal for larger purchases

**Process:**
1. Select Crypto payment option
2. System generates unique payment address
3. User sends exact USDT amount via TRC20
4. Payment confirmed automatically (5-30 min)
5. Service activated upon confirmation

**Network Details:**
- **Network:** TRC20 (Tron)
- **Currency:** USDT (Tether)
- **Confirmations Required:** 1-3
- **Address Format:** T-address (e.g., `TXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX`)

**Best for:**
- Large purchases (> $20)
- Crypto-native users
- Users in regions with limited payment options
- Privacy-conscious users

### Payment Method Comparison

| Feature | Telegram Stars | Crypto (USDT) |
|---------|----------------|---------------|
| Speed | Instant | 5-30 minutes |
| Minimum | 1 ⭐ (~$0.008) | $1.00 |
| Maximum | ~$10,000 | ~$10,000 |
| Fees | Telegram fees apply | Network fee only |
| Refunds | Via Telegram | Manual process |
| Privacy | Telegram has data | Pseudonymous |
| Availability | Telegram users only | Anyone with crypto |

---

## Calculations & Formulas

### Currency Conversion

```python
# Stars to USDT
usdt = stars / 120

# USDT to Stars
stars = usdt * 120
```

### Package Pricing

```python
# Price per GB
price_per_gb = package_price_usdt / total_data_gb

# Discount vs consumption billing
discount_percent = ((consumption_price - package_price) / consumption_price) * 100
# Where consumption_price = base_gb * 0.50
```

### Data with Bonuses

```python
# Total data with bonuses
total_data_gb = base_gb * (1 + (base_bonus + user_bonuses) / 100)

# Example: Premium package (120 GB) with 20% base bonus + 10% loyalty
total_data_gb = 120 * (1 + (20 + 10) / 100) = 120 * 1.30 = 156 GB
```

### Subscription Monthly Equivalent

```python
# Monthly equivalent price
monthly_price = total_price / duration_months

# Example: 6-month plan at $12.99
monthly_price = 12.99 / 6 = $2.17/month

# Effective monthly with bonus
effective_monthly = monthly_price * (1 - bonus_percent / 100)
```

### Consumption Billing

```python
# Cost calculation
cost_usd = data_consumed_gb * PRICE_PER_GB
# Where PRICE_PER_GB = 0.50

# MB calculation
cost_usd = data_consumed_mb * (PRICE_PER_GB / 1024)
# = data_consumed_mb * 0.00048828125
```

### Break-Even Analysis

```python
# Break-even point: when package becomes cheaper than consumption
break_even_gb = package_price / PRICE_PER_GB

# Example: Premium package ($12.00)
break_even_gb = 12.00 / 0.50 = 24 GB

# If you consume > 24 GB, package is cheaper
# If you consume < 24 GB, consumption billing is cheaper
```

### ROI for Subscription vs Packages

```python
# Daily cost
daily_cost = package_price / validity_days

# Example: Premium package
daily_cost = 12.00 / 35 = $0.34/day

# Subscription daily cost (6-month)
daily_cost = 12.99 / 180 = $0.07/day

# Savings with subscription
savings_percent = ((package_daily - subscription_daily) / package_daily) * 100
= ((0.34 - 0.07) / 0.34) * 100 = 79% savings
```

---

## Quick Reference

### Most Popular Packages

| Rank | Package | Why Popular |
|------|---------|-------------|
| 1 | **Premium (120 GB)** | Best value for heavy users |
| 2 | **Estándar (30 GB)** | Affordable entry point |
| 3 | **Ilimitado (200 GB)** | Maximum data, lowest per-GB cost |

### Most Popular Subscriptions

| Rank | Plan | Why Popular |
|------|------|-------------|
| 1 | **6 Months** | Best balance of savings and commitment |
| 2 | **3 Months** | Low commitment, decent discount |
| 3 | **12 Months** | Maximum savings for long-term users |

### Best Value Recommendations

| User Profile | Recommended Option | Reason |
|--------------|-------------------|---------|
| Light user (< 5 GB/month) | Consumption Billing | Pay only for what you use |
| Regular user (5-20 GB/month) | Estándar/Avanzado Package | 67-73% savings |
| Heavy user (20-100 GB/month) | Premium Package | 80% savings |
| Power user (100+ GB/month) | Ilimitado Package | 85% savings, no overage |
| Long-term user | 6-Month Subscription | Unlimited data, $1.89/month |

---

## Appendix: Constants Reference

### Code Constants

```python
# usipipo_commons/constants/plans.py
FREE_GB = 5.0                    # Free monthly data per key
FREE_KEYS_LIMIT = 2              # Default keys for free users
PRICE_PER_GB = 0.50              # Consumption billing price (USD)
BILLING_CYCLE_DAYS = 30          # Billing cycle duration

# usipipo_commons/constants/__init__.py
STARS_PER_USDT = 120             # Exchange rate

# Backend configuration
CONSUMPTION_PRICE_PER_GB_USD = 0.25  # Alternative consumption price (backend)
```

### API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/data-packages` | GET | List available data packages |
| `/subscriptions/plans` | GET | List available subscription plans |
| `/consumption/activate` | POST | Activate consumption billing |
| `/consumption/status` | GET | Get consumption status |

---

**Document Version:** 1.0.0  
**Maintained by:** uSipipo Team  
**Contact:** support@usipipo.com
