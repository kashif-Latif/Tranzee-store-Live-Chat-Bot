# 💄 Trixi Bot — Trenzee Cosmetics AI Chat Assistant

An AI-powered sales chatbot embedded on the **Trenzee Cosmetics** Shopify store
(makeup, skincare, haircare, beauty devices). It chats with customers, searches live
products, handles order tracking, takes orders via WhatsApp, and hands off to a human agent.

---

## What it does

- **AI conversation** in the customer's own language (English / Urdu / Roman Urdu).
- **Live product search** — finds real products with current prices, stock, discounts, and images.
- **Order tracking** — by order number (#TRZ), tracking ID, phone, or email. Shows order name,
  total, payment status, fulfilment status, courier, and tracking number, with a one-tap
  **Track your order** button to the correct courier.
- **Place an order** — a form that sends the order to your WhatsApp (no Size field — cosmetics).
- **Talk to an agent** — WhatsApp handoff (Mon–Sat, 10am–6pm, replies in 1–2 hours).

---

## Architecture (6 parts)

| # | Part | Where it lives | Holds keys? |
|---|------|----------------|-------------|
| 1 | **Widget** (`trixi-chat-widget.liquid`) | Shopify theme snippet `chat-widget` | ❌ No |
| 2 | **Backend** (`api/chat.js`) | This repo → Vercel | ✅ Yes (env vars) |
| 3 | **AI providers** | Groq + Gemini (via backend) | — |
| 4 | **Shopify Admin API** | Live price/stock/orders | — |
| 5 | **Shopify `suggest.json`** | Storefront search engine (no key) | — |
| 6 | **WhatsApp** | Human handoff / order confirmation | — |

**Message flow:** customer types → widget → backend → AI understands intent →
`suggest.json` finds products → Admin API verifies live price/stock → AI writes one-line reply →
widget shows product cards + reply.

---

## AI fallback chain (self-healing)

Tried in order; drops to the next automatically on rate-limit (429) or error:

```
Groq llama-3.3-70b  →  Groq llama-3.1-8b-instant  →  Gemini 3.6 Flash  →  Gemini 3.5 Flash-Lite  →  keyword fallback
```

Re-evaluated fresh on every message, so it recovers the moment Groq's limit resets.
If all AIs are down, it still shows live products with a simple reply (never a hard error).

---

## Search strategy

- **Product search** — Shopify `suggest.json` (the store's own search engine) ranks results;
  each is matched back to live Admin data for accurate price/stock/image.
  - Specific query (a product name) → the accurate product, with its price stated if asked.
  - Broad query ("lipstick", "serum") → many, for browsing.
  - "all products" / "cosmetics" / "everything" → a catalog sample.
  - Typos are cleaned by the AI first ("foundaton" → "foundation").
- **Discount** — "sale / discount / deal" → discounted items, highest % first.

> Search relies on Shopify's storefront engine + live Admin data. (A cosmetics synonym map
> can be added later from the Trenzee product CSV for extra typo/synonym smarts.)

---

## Order tracking (real courier)

- Enter an **order number**, **tracking ID**, phone, or email.
- The backend looks up the order in Shopify, reads the **real courier** (PostEx / OwnExpress)
  and tracking number from the fulfilment, and returns a single **Track your order** button.
- If Shopify has a direct tracking deep-link, the button opens straight to live tracking;
  otherwise the tracking number is shown with a **Copy** button and the correct courier page.

---

## Repository structure

```
.
├── api/
│   └── chat.js          ← backend (MUST be at api/chat.js)
├── package.json
├── .gitignore
└── .gitattributes
```

> The widget (`trixi-chat-widget.liquid`) does **not** go in this repo —
> it is pasted into the Shopify theme.

---

## Environment variables (set in Vercel → Settings → Environment Variables)

| Variable | Value / Notes |
|----------|---------------|
| `GROQ_API_KEY` | Groq API key |
| `GEMINI_API_KEY` | Gemini API key (optional but recommended) |
| `SHOPIFY_STORE` | `f9ikjt-d0.myshopify.com` |
| `SHOPIFY_ADMIN_TOKEN` | Shopify Admin API access token (`shpat_…`) |
| `WHATSAPP_NUMBER` | `923226885324` |
| `PUBLIC_DOMAIN` | `https://www.trenzeecosmetics.com` |
| `ALLOWED_ORIGIN` | `https://www.trenzeecosmetics.com` (use `*` while testing) |
| `GROQ_MODELS` | *(optional)* comma-separated model override |
| `GEMINI_MODELS` | *(optional)* comma-separated model override |

**Shopify Admin token scopes required:** `read_products`, `read_inventory`, `read_orders`, `read_customers`.

> 🔒 Keys live **only** in Vercel — never in the code or the widget.

---

## Deploy

1. **Backend:** push this repo to GitHub → import into Vercel (a **separate** project from Bee Bot)
   → add env vars → Deploy. Endpoint: `/api/chat`.
2. **Widget:** in Shopify → Online Store → Themes → Edit code → Snippets → add
   `chat-widget` → paste `trixi-chat-widget.liquid` and set:
   ```js
   var VERCEL_URL = "https://tranzee-store-live-chat-bot.vercel.app/api/chat";
   ```
   Then in `theme.liquid`, before `</body>`, add:
   ```liquid
   {% render 'chat-widget' %}
   ```
3. Hard-refresh the site (Ctrl+Shift+R).

**Health check:** open `https://tranzee-store-live-chat-bot.vercel.app/api/chat`
→ "Method not allowed" = alive.

---

## Notes

- Product data is fetched **live** from Shopify every time — nothing is stored.
- Conversations persist only in the customer's browser (localStorage), cleared on "New chat".
- Theme: pink + light cream + black; lipstick logo; **Live products** indicator dot in the header.
- Order prefix: **#TRZ**. Carriers: **PostEx** and **OwnExpress**.
- This bot is fully separate from **Bee Bot** (Little Minors) — different Vercel project & store keys,
  so the two never share products.
