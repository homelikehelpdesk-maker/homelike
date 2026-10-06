# Homelike Spices

Storefront (Vite + React + Tailwind v4) backed by **Supabase** (Postgres +
Auth + Edge Functions). Customers order through an on-site checkout, orders
land in Postgres, and every new order **emails the owner** (plus an optional
customer confirmation) automatically. The owner reviews and updates orders in
the admin console at `#/admin`.

---

## 1 · Create the Supabase project

1. New project at <https://supabase.com> → copy the **Project URL** and
   **anon public key** (Project Settings → API).
2. Paste them into `.env` (this file is git-ignored):

```env
VITE_SUPABASE_URL="https://YOUR-PROJECT-REF.supabase.co"
VITE_SUPABASE_ANON_KEY="eyJhbGciOi..."
```

These are **public by design** — Row Level Security, not secrecy, is what
protects the data. Never put the `service_role` key in the frontend.

## 2 · Create the orders table + security policies

Supabase → SQL Editor → run:

```sql
create table if not exists public.orders (
  id bigint generated always as identity primary key,
  items jsonb not null,
  total integer not null,
  customer_name text not null,
  customer_phone text not null,
  customer_address text not null,
  customer_email text,
  status text not null default 'new',
  created_at timestamptz not null default now()
);

alter table public.orders enable row level security;

-- anyone (including anonymous visitors) can PLACE an order, nothing more
create policy "anyone can place an order"
  on public.orders for insert to anon with check (true);

-- only signed-in admins can READ orders
create policy "admins can read orders"
  on public.orders for select to authenticated using (true);

-- only signed-in admins can UPDATE status (new → confirmed → delivered)
create policy "admins can update status"
  on public.orders for update to authenticated using (true) with check (true);
```

## 3 · Create the admin user

Supabase → Authentication → Users → **Add user** → choose an email + password
(this is what you sign in with at `#/admin`). Mark **Auto Confirm User** on.

## 4 · Email notifications (Resend + Edge Function + Webhook)

1. Free account at <https://resend.com> → copy an API key. Until you verify a
   domain, you can send from `onboarding@resend.dev` to **your own inbox only**
   — perfect for owner notifications. Add both emails to `allowed` testing
   recipients under Resend if you want customer confirmations pre-domain.
2. Deploy the function (Supabase CLI: `npm i -g supabase`):

```bash
supabase login
supabase link --project-ref YOUR-PROJECT-REF
supabase functions deploy order-email
supabase secrets set \
  RESEND_API_KEY="re_xxxxxxxx" \
  OWNER_EMAIL="homelikespices@gmail.com" \
  WEBHOOK_SECRET="$(openssl rand -hex 24)"
# optional, once a domain is verified in Resend:
# supabase secrets set FROM_EMAIL="Homelike Spices <orders@yourdomain.com>"
```

3. Supabase → Database → **Webhooks** → Create:
   - Table: `orders`, Events: **INSERT**
   - Type: HTTP Request, Method: POST
   - URL: `https://YOUR-PROJECT-REF.functions.supabase.co/order-email`
   - HTTP header: `x-webhook-secret` = the same `WEBHOOK_SECRET` you set above

Done — each new order emails the owner a formatted receipt and the customer a
confirmation (when they leave an email).

> Simpler alternative without webhooks: call the function URL directly from the
> client after a successful insert, keeping the `x-webhook-secret` header
> server-side is then impossible — the webhook route above is recommended.

## 5 · Run & deploy

```bash
npm install
npm run dev        # http://localhost:5173
```

The site is fully static — host `npm run build`'s `dist/` anywhere (Netlify,
Vercel, GitHub Pages…) and set the two `VITE_*` vars in that host's env
dashboard. The backend lives entirely in Supabase; no server code is needed on
your host.

## 6 · Admin console

`/#/admin` → sign in with the Supabase Auth user from step 3. See the latest
200 orders, call customers with one tap, and advance each order's status
(**new → confirmed → delivered**).
