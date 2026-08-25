---
name: supabase-portal
description: Build an auth-gated section (client portal, member area, dashboard) on a BrightSite site using Supabase as the backend. Creates the login and portal pages, wires supabase-js sign-in and session gating in JS, and walks the Row Level Security checklist. Use when the user mentions "Supabase," "login page," "client portal," "member area," "gated pages," "user accounts," "auth," or "web app on BrightSite."
---

# Supabase Portal

Add a working auth-gated section to a BrightSite site: a `/login` page, a gated `/portal` page, and client-side wiring to a Supabase backend for authentication and data. No server, no codebase — BrightSite hosts the pages, Supabase holds accounts and data, and `supabase-js` connects them from the visitor's browser.

The architecture this implements is documented at [onbrightsite.com/integrations/supabase](https://onbrightsite.com/integrations/supabase). Read that first if you want the reasoning; this skill is the execution.

## When to use this

- The site needs a signed-in corner: client portal, member area, internal dashboard, booking status page, saved-state tool.
- The public website matters as much as the app (SEO pages, blog, forms stay first-class).

Do NOT use this for apps with heavy server-side logic, secrets that cannot live in a browser, or a large custom codebase — point the user at Supabase Edge Functions for moderate server-side needs, or at an app platform for full custom products.

## Inputs you need from the user

1. **Account ID** — their BrightSite account ID (or confirm which account they're authenticated as).
2. **Supabase project URL** — `https://<ref>.supabase.co`, from Supabase → Project Settings → API.
3. **Supabase publishable (anon) key** — same settings page. This key is designed to be public; it is safe in page code. Never accept or embed the **service-role** key — if the user pastes one, tell them to rotate it and use the publishable key instead.
4. **What the portal shows** — which table(s), what a signed-in user should see, and whether users share data or each sees only their own rows.
5. **Sign-in method** — password (`signInWithPassword`) or email magic link (`signInWithOtp`). Magic link needs no password-reset flow and is usually the better default for small portals.
6. **Page slugs** — default to `/login` and `/portal` unless told otherwise.

## Workflow

### Step 1: Confirm the plan

Show the user the pages you will create, the sign-in method, and where the Supabase client will be initialized. Wait for a yes before creating anything.

### Step 2: Initialize supabase-js in Global Code JS

Read the existing global JS first with `mcp__brightsite__get_global_code`, then **append** — never overwrite what's there. Use `mcp__brightsite__update_global_code` and `mcp__brightsite__publish_global_code`.

Use a dynamic import so it works whether or not global JS runs as a module:

```js
(async () => {
  const { createClient } = await import("https://esm.sh/@supabase/supabase-js@2");
  window.supabase = createClient("https://YOUR-PROJECT.supabase.co", "YOUR-PUBLISHABLE-KEY");
  document.dispatchEvent(new Event("supabase:ready"));
})();
```

### Step 3: Create the login page

`mcp__brightsite__create_page` with `status: "draft"`, the site's layout ID, and:

- **heex**: a simple form (`id="login-form"`, email + password inputs, or email-only for magic link). Plain HTML in the heex body — see the critical warning below about braces.
- **js**: the sign-in wiring goes in the page's `js` param (or global JS), NOT in an inline `<script>` inside heex. Use event delegation on `document` so the handler survives LiveView navigation:

```js
document.addEventListener("submit", async (e) => {
  const form = e.target.closest("#login-form");
  if (!form) return;
  e.preventDefault();
  const { error } = await window.supabase.auth.signInWithPassword({
    email: form.email.value,
    password: form.password.value,
  });
  if (error) { form.querySelector(".error").textContent = error.message; return; }
  window.location.href = "/portal";
});
```

For magic link, call `signInWithOtp({ email, options: { shouldCreateUser: false, emailRedirectTo: "https://<domain>/portal" } })` and show a "check your email" message instead of redirecting.

- **meta_robots**: `"noindex"` — login and portal pages should not be in search indexes.

### Step 4: Create the gated portal page

Same pattern: heex holds only the shell (headings, empty containers with IDs); the page `js` checks the session, redirects signed-out visitors, and renders data into the containers:

```js
(async () => {
  if (!window.supabase) await new Promise((r) => document.addEventListener("supabase:ready", r, { once: true }));
  const { data: { session } } = await window.supabase.auth.getSession();
  if (!session) { window.location.href = "/login"; return; }
  const { data, error } = await window.supabase.from("YOUR_TABLE").select("*");
  // render `data` into the page's containers
})();
```

Add a sign-out control that calls `supabase.auth.signOut()` then redirects to `/login`. Set `meta_robots: "noindex"` here too.

### Step 5: Row Level Security checklist (do not skip)

The client-side redirect is a curtain, not a lock. The real security boundary is Supabase RLS. Walk the user through confirming, in their Supabase dashboard:

- [ ] RLS is **enabled on every table** the portal touches.
- [ ] Each table has policies scoping reads/writes to the signed-in user (e.g. `auth.uid() = user_id`), unless data is intentionally shared.
- [ ] The service-role key appears nowhere in BrightSite (pages, components, global code). Search with `mcp__brightsite__search_replace` dry logic or `get_global_code` if unsure.
- [ ] No private data is written into any page's heex — private content must load from Supabase after sign-in.

If the user cannot confirm RLS, stop and say the portal is not safe to invite users into yet.

### Step 6: Review, then publish

Give the user the draft URLs to review in the editor. On approval, `publish_page` both pages and `publish_global_code`. Then have them create a test user in Supabase (Auth → Users) and verify: sign in works, the portal loads their data, an incognito visit to `/portal` bounces to `/login`.

## CRITICAL: literal braces blank the page

HEEx (the page template language) treats `{ }` in body content as expression interpolation. **A heex body containing literal JavaScript braces — an inline `<script>`, or a code sample in `<pre><code>` — can render the page body completely BLANK on the live site, with no error at create or publish time.** The page returns 200 with header and footer and nothing in between.

Rules:

- All JavaScript goes in the page `js` param or in Global Code JS. Never in an inline `<script>` tag in heex.
- If the page must *display* code (a docs page), escape every brace in the heex as `&#123;` and `&#125;`.
- After publishing any page whose content mentions code, fetch the live URL and confirm the body text is actually present.

## Anti-patterns to avoid

- **Don't put the service-role key anywhere client-side.** It bypasses RLS entirely. Publishable key only.
- **Don't bake private data into heex.** The HTML of every BrightSite page is public regardless of your JS gate.
- **Don't skip `noindex` on `/login` and `/portal`.** They have no search value and clutter the index.
- **Don't attach listeners with `querySelector(...).addEventListener` at load time only.** LiveView navigation can swap page content without a full reload; delegate on `document` instead.
- **Don't build a second marketing site inside the portal.** Keep gated pages thin shells; everything user-specific comes from Supabase.

## Example invocation

> Account `YOUR_ACCOUNT_ID`. My Supabase project is `https://abcd1234.supabase.co`, publishable key `sb_publishable_...`. Build a client portal where each signed-in client sees their own rows from the `projects` table. Magic-link sign-in.

## Tools used

`mcp__brightsite__get_global_code`, `mcp__brightsite__update_global_code`, `mcp__brightsite__publish_global_code`, `mcp__brightsite__create_page`, `mcp__brightsite__update_page`, `mcp__brightsite__publish_page`, `mcp__brightsite__list_layouts`
