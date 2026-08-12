# Lindo's Kitchen

**Live site:** [lindos-kitchen.co.za](https://www.lindos-kitchen.co.za)

The website for **Lindo's Kitchen**, a kota/kitchen takeaway: a public menu
customers can browse and order from via **Call** or **WhatsApp**, plus a
hidden, password-protected owner dashboard for editing the menu — items,
prices, photos, and categories — without touching any code.

Built with **React + TypeScript + Tailwind CSS** (via Vite), **Supabase** as
the database/auth/file-storage backend, and deployed on **Vercel**.

---

## 1. How the site is put together

```
React + TypeScript (Vite)                  Supabase
──────────────────────────                 ─────────────────────
CustomerPage (/)              ⇄  useMenu        Postgres: menu_items
 ├─ Navbar, Hero, Footer       ⇄  useCategories   Postgres: categories
 ├─ CategoryTabs, MenuGrid,                       Storage: category-images
 │   MenuItemCard, ExtrasList
AdminPage (hidden route)      ⇄  useAuth         Auth: owner login
 ├─ AdminLogin
 └─ AdminDashboard
```

It's a single-page app (React Router) with two routes: the public **customer
page** and a **hidden owner dashboard**. Both talk to the same Supabase
project — the customer page only ever *reads* the menu; only a signed-in
owner can write to it. Three custom hooks (`useMenu`, `useCategories`,
`useAuth`) own all the Supabase calls, so the components themselves stay
pure UI.

## 2. The screens

| Screen | Where | Purpose |
|---|---|---|
| **Customer page** | `/` | Hero with the shop's address and order buttons, category tabs, and the menu list for whichever category is selected. |
| **Owner dashboard** | hidden route | Password-protected. Lets the owner manage categories (add/rename/reorder/delete, each with its own photo) and menu items (add/edit inline/delete), filtered by category. |

Menu content is **not hard-coded** — categories and items both live in
Supabase, so the owner can restructure the whole menu from the dashboard
without a code change or redeploy.

## 3. End-to-end flow

1. **Customer visits the site** — the menu loads from Supabase (or from a
   built-in local fallback list if Supabase is briefly unreachable, so the
   site never shows a blank page).
2. **Browsing** — the customer taps a category tab (each with its own
   photo) to see that category's items, then taps **WhatsApp** (opens a
   pre-filled chat with the shop) or **Phone** (`tel:` call) to order.
3. **Sharing** — the navbar's Share button uses the device's native share
   sheet on mobile, or copies the link on desktop.
4. **Owner signs in** — via the hidden dashboard route, using an email/
   password account created directly in Supabase (there's no public
   sign-up flow).
5. **Managing categories** — the owner can add a new category (with an
   optional photo upload), rename one, change its photo, delete it, or
   reorder categories with up/down arrows — the order set here is exactly
   what customers see as tabs.
6. **Managing items** — the owner filters the item table by category,
   edits any field inline (saves automatically on blur), adds new items,
   or deletes existing ones. On phones the table collapses into a stacked
   card layout instead of a cramped grid.
7. **Changes go live immediately** — there's no separate "publish" step;
   the customer page always reads the current state of the database.

## 4. Data & access model

- **Two Supabase tables:** `menu_items` (name, price, description, photo,
  category, sort order) and `categories` (name, photo, sort order).
- **Row Level Security** is enabled on both: anyone can *read* (needed for
  the public menu), but only an authenticated Supabase user (the owner) can
  *write*. There's no separate admin flag or role table — being logged in
  at all is what makes someone "the owner."
- **Photo uploads** for categories go to a public Supabase Storage bucket
  (`category-images`); item photos are set via a plain image URL field.
- **The dashboard route is intentionally unlisted** — it isn't linked
  anywhere on the public site, so the only way to reach it is knowing the
  URL directly.

## 5. Tech stack

- **Frontend:** React 19 + TypeScript, React Router, Vite
- **Styling:** Tailwind CSS
- **Backend:** Supabase (Postgres database, Auth, Storage) — no custom
  server; the frontend talks to Supabase directly
- **Hosting:** Vercel

## 6. Project structure

```
src/
├─ components/
│  ├─ Navbar, Hero, Footer               Public-site chrome
│  ├─ CategoryTabs, MenuGrid,
│  │  MenuItemCard, ExtrasList           Public menu display
│  ├─ AdminLogin, AdminDashboard         Owner dashboard
│  └─ LoadingScreen                      Brief splash on first load
├─ pages/
│  ├─ CustomerPage.tsx                   The "/" screen
│  └─ AdminPage.tsx                      The hidden dashboard screen
├─ hooks/
│  ├─ useMenu.ts                         Read/write menu_items
│  ├─ useCategories.ts                   Read/write categories + photo upload/reorder
│  └─ useAuth.ts                         Owner login/session state
├─ lib/
│  ├─ supabase.ts                        Supabase client
│  └─ contact.ts                         Shop phone/WhatsApp/address + link helpers
├─ data/menuData.ts                      Local fallback menu
└─ types.ts                              Shared TypeScript types
```

A couple of implementation details worth calling out:

- **Fully dynamic categories** — `MenuCategory` used to be a strict
  `'kota' | 'extra' | 'chicken'` union; since the dashboard now lets the
  owner type any category name, it's a plain `string`, and the tabs on the
  customer page are generated from whatever's in the `categories` table.
- **Optimistic UI updates** — dashboard edits (renaming a category,
  updating an item) update the screen immediately and roll back
  automatically if the Supabase write fails, rather than waiting on a
  round trip before showing the change.