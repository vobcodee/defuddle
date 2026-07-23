# Vercel Deployment Migration Guide

The `defuddle` website is currently architected specifically for **Cloudflare Workers**. Migrating this project to Vercel requires refactoring because it heavily relies on Cloudflare-exclusive infrastructure.

## 1. Cloudflare Dependencies to Refactor

To deploy to Vercel, you must replace the following Cloudflare APIs in `website/src/index.ts`:

*   **Cloudflare KV (`RATE_LIMIT`)**: Used for rate-limiting.
    *   *Vercel Equivalent*: **Vercel KV** (Redis) or `@vercel/edge` rate limiting.
*   **Durable Objects (`API_KEY_BALANCES`, `CHECKOUT_FULFILLMENTS`)**: Used for transactional state management for API keys and Stripe checkouts.
    *   *Vercel Equivalent*: **Vercel Postgres** or another database (e.g., Supabase, Neon) since Vercel lacks a direct Durable Objects equivalent.
*   **Cache API (`caches.default`)**: Used to cache static pages at the edge.
    *   *Vercel Equivalent*: Use standard `Cache-Control` HTTP headers (e.g., `s-maxage`) to leverage Vercel's Edge Network.

## 2. Migration Strategy

### Step 1: Create a Vercel Entry Point
Create a Vercel Edge Function wrapper (e.g., `api/index.ts`) that mimics the Cloudflare `fetch` handler.

### Step 2: Refactor State Management
Abstract the rate limiting and billing logic. For example, change `checkRateLimit` to use Redis:
```typescript
import { kv } from '@vercel/kv';
// Replace Cloudflare KV logic with Vercel KV
```

### Step 3: Add `vercel.json`
Configure Vercel to route traffic to your new Edge Function:
```json
{
  "version": 2,
  "rewrites": [
    { "source": "/(.*)", "destination": "/api/index" }
  ]
}
```

### Step 4: Environment Variables
Map your Cloudflare environment variables (`STRIPE_SECRET_KEY`, etc.) to Vercel environment variables in the Vercel dashboard.

## Recommendation

Because the core billing and rate-limiting features are tightly coupled to **Durable Objects**, a direct "lift and shift" to Vercel is not possible without rewriting the `index.ts` logic. 

**If your goal is just to host the landing page and playground without the paid API features:**
You can strip out the Durable Objects and Stripe logic from `website/src/index.ts`, adapt the caching to use headers, and deploy the remaining logic as a standard Vercel Edge Function.
