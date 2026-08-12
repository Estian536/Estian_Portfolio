# Mae's Kitchen

A Portuguese takeaway's website: a landing page with contact info and order
links, a printed-menu-style menu page, and a hidden, password-protected
dashboard where the owner can add, edit, and delete menu items — without
touching any code.

Built with **React + TypeScript + Tailwind CSS** (via Vite), **Supabase** as
the database/auth backend, and deployable to **Vercel**.

---

## 1. How the site is put together

```
React + TypeScript (Vite)                  Supabase
──────────────────────────                 ─────────────────────
LandingPage (/home)                              (no DB — static contact info)
MenuPage (/menu)              ⇄  fetch on load    Postgres: menu_items
AdminPage (hidden route)      ⇄  fetch + writes    Postgres: menu_items
 ├─ AdminLogin                ⇄  Supabase Auth      Auth: owner login
 └─ AdminMenuManager
```

It's a single-page app (React Router) with three routes: a **landing page**,
a **public menu page**, and a **hidden owner dashboard**. There's a single
`menu_items` table — no separate categories table like some of the other
sites in this series — so "category" is just a text field on each item, and
the list of categories shown anywhere is derived from whatever's actually
in use.

## 2. The screens

| Screen | Where | Purpose |
|---|---|---|
| **Landing page** | `/home` (and `/`, which redirects there) | Shop name, banner photo, a big **MENU** button, and a contact card (location, phone, WhatsApp, opening hours). A floating share button opens the native share sheet on mobile or copies the link on desktop. |
| **Menu page** | `/menu` | The full printed-menu-style listing, grouped by category, laid over a background image. Mobile shows a simple stacked list; desktop balances categories across 5 columns automatically. |
| **Owner dashboard** | hidden route | Password-protected. Add a new item (pick or type a category, name, optional price), and edit/delete/rename any existing item or category inline. |

## 3. End-to-end flow

1. **Customer lands on the site** — sees the shop banner, hours, and
   contact options, and taps **MENU**.
2. **Browsing the menu** — items load from Supabase, grouped by category.
   An item with no price shows **N/A** instead (used for sold-out or
   currently-unavailable dishes rather than deleting them outright).
3. **Ordering** — the customer taps **Chat on WhatsApp** (opens a
   pre-filled delivery-order message) or calls the kitchen directly via the
   phone link on the landing page.
4. **Sharing** — the floating share button on the landing page uses the
   device's native share sheet, or copies the URL on desktop.
5. **Owner signs in** — via the hidden dashboard route, using an email/
   password account created directly in Supabase (no public sign-up).
6. **Adding an item** — the owner picks an existing category from a
   dropdown or types a brand-new one on the fly, enters a name and
   optional price, and submits.
7. **Editing** — any item can be edited in place (category/name/price) or
   deleted with a confirmation prompt. A whole category can be renamed in
   one action, which updates every item under it.
8. **Changes go live immediately** — there's no separate "publish" step;
   the menu page always reads the current state of the database.

## 4. Data & access model

- **One Supabase table:** `menu_items` (`id`, `category`, `name`, `price`).
  `price` is nullable — `null` means "N/A" (sold out / unavailable) rather
  than the item being removed.
- Categories aren't a separate table here — they're just the distinct
  `category` values across items, grouped and sorted client-side.
- The dashboard currently reads/writes without table-level Row Level
  Security policies scoping who can do what beyond being signed in — access
  control is "you have a Supabase login or you don't."
- **The dashboard route is intentionally unlisted** — it isn't linked
  anywhere on the public site, so the only way to reach it is knowing the
  URL directly.

## 5. Tech stack

- **Frontend:** React 19 + TypeScript, React Router, Vite
- **Styling:** Tailwind CSS (plus a hand-written `<style>` block for the
  landing page's card/action-sheet layout)
- **Backend:** Supabase (Postgres database, Auth) — no custom server; the
  frontend talks to Supabase directly
- **Hosting:** Vercel-ready (`vercel.json` included for SPA routing)

## 6. Project structure

```
src/
├─ components/
│  ├─ AdminLogin.tsx                    Owner sign-in form
│  ├─ AdminMenuManager.tsx              Add / edit / delete / rename items & categories
│  ├─ CategorySelect.tsx                Category dropdown with inline "add new" option
│  ├─ CategorySection.tsx               One category block on the menu page
│  └─ MenuItemRow.tsx                   Single item row (name + price)
├─ pages/
│  ├─ LandingPage.tsx                   "/home" — banner, contact card, share button
│  ├─ MenuPage.tsx                      "/menu" — full menu, grouped & column-balanced
│  └─ AdminPage.tsx                     Hidden dashboard route (auth-gated)
├─ utils/
│  ├─ groupByCategory.ts                Groups a flat item list into { category: items[] }
│  └─ assignColumns.ts                  Distributes categories across N desktop columns
├─ lib/supabase.ts                      Supabase client
└─ types.ts                             Shared MenuItem type
```

A couple of implementation details worth calling out:

- **Column-balancing algorithm** (`assignColumns.ts`) — on desktop, the
  menu is laid out in 5 columns. Rather than splitting categories evenly by
  *count*, it sorts categories largest-first and always drops the next one
  into the currently lightest column (weighted by item count, with a
  fudge-factor for each category's header), so columns end up visually even
  in height even though categories vary wildly in size.
- **Price as "N/A"** — `price: number | null` lets the menu show an item as
  unavailable without deleting it or faking a price, which matters for a
  kitchen where dishes go on and off the board day to day.
- **Inline category creation** — `CategorySelect` doubles as both a picker
  and a free-text input: choosing "+ Add new category" swaps the dropdown
  for a text field, so the owner never has to manage categories as their
  own separate step.