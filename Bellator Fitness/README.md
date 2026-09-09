Bellator Fitness
A gym's website: a single-page public site with about, class schedule, trainers, gallery, pricing, and contact sections, plus a hidden, password-protected admin dashboard where the owner can update every one of those sections — without touching any code.

Built with React + TypeScript + Tailwind CSS (via Vite) on the frontend, and a plain PHP + MySQL API on the backend, deployed to HostAfrica shared hosting with GitHub Actions handling every push.

1. How the site is put together
React + TypeScript (Vite)                   PHP API (bellator-api)              MySQL
──────────────────────────                  ─────────────────────              ─────────────────────
PublicSite (/)                    ⇄  GET   about.php / classes.php /            about_content
 ├─ About, Classes, Trainers,             trainers.php / pricing.php /          classes
 │  Gallery, Pricing, Contact             gallery.php / contact.php             trainers
 │  (each section fetches its                                                   pricing_plans / pricing_features
 │  own data independently)                                                     gallery_images
AdminDashboard (hidden route)     ⇄  POST/PUT/DELETE  <section>_create.php /    contact_info
 ├─ AdminLogin                            _update.php / _delete.php             admin_users
 └─ AdminOverview / AdminAbout /   ⇄  Session auth     login.php /
    AdminClasses / AdminTrainers /        logout.php / check_session.php
    AdminPricing / AdminGallery /
    AdminContact
It's a single-page app (React Router) with one public route and one nested, auth-gated admin route. There's no shared backend framework — each PHP file is a small, single-purpose endpoint that talks to MySQL directly via PDO — so every content type (classes, trainers, pricing, gallery, contact, about) has its own read endpoint and, for the admin side, its own create/update/delete endpoints.

2. The screens
Screen	Where	Purpose
Public site	/	One scrolling page: About blurb, weekly class schedule, trainer bios with photos, a photo gallery with a "show more" toggle, pricing plans with feature lists, and a contact/location block — plus floating social links and a sticky navbar.
Admin dashboard	hidden route (/bf-admin-2436)	Password-protected, sidebar-driven dashboard with one screen per content type: Overview (counts), About Content, Classes, Trainers, Pricing, Gallery, and Contact & Location. Each screen lets the owner add, edit, delete, and reorder items in that section.
3. End-to-end flow
Visitor lands on the site — the navbar, About section, and the rest load their content from the API as soon as the page mounts; nothing is hardcoded in the React components beyond a loading/fallback state.
Browsing — classes are grouped by day and sorted into calendar order; the gallery loads images, checks each one's orientation client-side, and shows a limited preview with a "view more" expansion; pricing plans list their features underneath a price and billing period.
Joining — a "Join Now" link (configurable from the Contact admin screen) is used across the Hero and Pricing sections instead of being hardcoded, so the owner can point it at a signup form or WhatsApp link without a code change.
Owner signs in — via the hidden dashboard route, using a username/password stored as a bcrypt hash in the admin_users table (checked with PHP's password_verify). A PHP session cookie keeps them logged in.
Editing content — each admin screen (About, Classes, Trainers, Pricing, Gallery, Contact) calls its own create/update/delete PHP endpoint; every write endpoint is gated behind require_auth.php, which checks the session before doing anything.
Uploading photos — trainer photos and gallery images go through upload.php (single) or gallery_batch_upload.php (up to 20 at once), which validates file type/size, resizes anything wider than 1200px, flattens it to a JPEG, and saves it under bellator-api/uploads.
Gallery housekeeping — gallery.php quietly runs a once-a-day cleanup (gallery_cleanup.php) that deletes gallery images older than 3 months and their files, and the batch upload endpoint enforces a 50-photo total cap so the gallery doesn't grow unbounded.
Changes go live immediately — there's no separate "publish" step; every public section reads the current state of the database on each page load.
Deploying — pushing to main on either repo (bellator-fitness or bellator-api) triggers a GitHub Actions workflow that builds (frontend only) and FTP-deploys straight to HostAfrica; no manual File Manager/FTP step is needed.
4. Data & access model
Six content tables, one per section: about_content, classes, trainers, pricing_plans (+ pricing_features), gallery_images, contact_info — plus admin_users for login.
Ordering is explicit: classes, trainers, pricing_plans, and gallery_images all carry a display_order column that the admin screens can rewrite, rather than relying on insertion order or ID.
price on pricing_plans and the class schedule are read-only from the public side — GET endpoints (about.php, classes.php, etc.) have no auth check at all, since they only ever expose already-public marketing content.
Every write endpoint (the _create.php / _update.php / _delete.php files, plus upload.php and stats.php) requires require_auth.php to pass, which checks for an admin_id in the PHP session — "you're logged in as the owner or you get a 401."
CORS is locked to the production domain: cors.php only allows requests from the live site's origin and enables credentials, so the API can't be called cross-origin from anywhere else.
The admin route is intentionally obscure (a non-guessable slug) rather than linked anywhere on the public site, so the main gate against outsiders is the login form and the unlisted URL together.
5. Tech stack
Frontend: React 19 + TypeScript, React Router, Vite
Styling: Tailwind CSS, with the gym's logo watermarked as a fixed low-opacity background
Backend: PHP (no framework) + MySQL via PDO, running on HostAfrica shared hosting; PHP sessions for auth, GD library for image resizing
Hosting/CI: HostAfrica for both the static frontend build and the PHP API; GitHub Actions (FTP-Deploy-Action) auto-deploys both repos on every push to main
6. Project structure
bellator-fitness/               (React frontend)
├─ src/
│  ├─ components/
│  │  ├─ auth/ProtectedRoute.tsx         Redirects to login if no active session
│  │  ├─ layout/
│  │  │  ├─ Navbar.tsx, Footer.tsx        Site chrome
│  │  │  ├─ AdminSidebar.tsx              Admin nav + logout
│  │  │  └─ FloatingSocials.tsx           Floating social/contact links
│  │  ├─ sections/
│  │  │  ├─ About.tsx, Classes.tsx, Trainers.tsx,
│  │  │  │  Gallery.tsx, Pricing.tsx, Contact.tsx, Hero.tsx
│  │  │  │                                 Each fetches its own data from the API
│  │  │  └─ ui/SectionHeading.tsx
│  ├─ pages/
│  │  ├─ public/PublicSite.tsx            Assembles all public sections
│  │  └─ admin/
│  │     ├─ AdminLogin.tsx, AdminDashboard.tsx, AdminOverview.tsx
│  │     └─ AdminAbout / AdminClasses / AdminTrainers /
│  │        AdminPricing / AdminGallery / AdminContact.tsx
│  ├─ context/AuthContext.tsx             Session state + login/logout calls
│  └─ utils/imageURL.ts                   Resolves relative upload paths to full URLs
│
bellator-api/                   (PHP backend)
├─ db.php, cors.php, require_auth.php     Shared DB connection, CORS, and auth guard
├─ login.php, logout.php, check_session.php
├─ about.php, about_update.php
├─ classes.php, classes_create.php, classes_update.php, classes_delete.php
├─ trainers.php, trainers_create.php, trainers_update.php, trainers_delete.php
├─ pricing.php, pricing_create.php, pricing_update.php, pricing_delete.php
├─ gallery.php, gallery_create.php, gallery_update.php,
│  gallery_delete.php, gallery_batch_upload.php, gallery_batch_delete.php,
│  gallery_cleanup.php
├─ contact.php, contact_update.php
├─ upload.php, image_helper.php            Single-image upload + resize/flatten to JPEG
└─ stats.php                               Counts per section, for the Overview screen
A couple of implementation details worth calling out:

One endpoint per verb, per section — rather than a REST framework, each content type gets its own small, explicit PHP file (classes.php to read, classes_create.php / classes_update.php / classes_delete.php to write). It's more files, but each one does exactly one query and is easy to reason about or lock down individually.
Image pipeline (image_helper.php) — every uploaded photo is decoded, downscaled to a max width of 1200px if larger, flattened onto a white background, and re-encoded as an 82%-quality JPEG before being saved with a unique filename. This keeps upload sizes predictable regardless of what a phone camera or PNG screenshot throws at it.
Self-cleaning gallery — gallery.php checks once a day (via a timestamp marker file) whether any gallery image is older than 3 months and, if so, deletes both the database row and the file on disk. Combined with the 50-photo cap in gallery_batch_upload.php, this keeps the gallery from growing forever without the owner having to manually prune it.
Deploy-on-push — both repos ship a GitHub Actions workflow that FTP-deploys on every push to main: the frontend workflow runs npm run build and uploads dist/, while the API workflow uploads the PHP files as-is (excluding uploads/, .git/, and .github/ so user-uploaded photos and repo metadata are never touched by a deploy).
