# kpground

## What this is

The landing page for a family site built for my daughter — a gateway to other things hosted for her, currently games. Lives at `kpground.com`. This repo is the landing page itself; each game (e.g. `kattrap`, sibling repo `../kat_trap`) is its own separate repo/app.

## Hosting architecture

Three providers, each with one job. Decided 2026-07-28 after comparing Heroku (no free dynos since Nov 2022), Cloudflare Workers (would require learning JS/TS + a new sandboxed runtime — ruled out for now), and Render+Neon (stays in frameworks already known).

| Piece | Provider | Why |
|---|---|---|
| DNS | **Cloudflare** (free) | `kpground.com`'s nameservers point here. Lets any subdomain (`kattrap.kpground.com`, etc.) be routed to whatever host that game actually uses, independent of where the landing page lives. Domain itself is registered through Squarespace Domains (it migrated there from Google Domains in 2023 — Google exited the registrar business) — only the *nameservers* point to Cloudflare, registration stays with Squarespace. |
| Compute | **Render** (free tier to start) | Landing page: free **Static Site** (no backend yet — see below). Each game: its own **Web Service** running Rails or Django, frontend assets and backend served together as one app. |
| Database | **Neon** (free, does not expire) | Explicitly *not* Render's own free Postgres — that auto-deletes after 30 days + a 14-day grace period unless upgraded to paid, which is a real trap for anything storing actual user accounts. Neon is still plain Postgres, so ActiveRecord/Django ORM connects the same way — just a different `DATABASE_URL`. |

## Auth (build when needed, not yet)

The landing page will eventually need a backend + accounts; every game will too. Plan, so it isn't rebuilt N times per game:

- **Landing page becomes the identity provider.** Normal framework auth — Devise (Rails) or `django.contrib.auth` (Django). Owns the shared `users` table in Neon.
- **Games on a `kpground.com` subdomain get SSO for free**: a session cookie scoped to `.kpground.com` is sent to every subdomain automatically. Log in once on the landing page, every game on a subdomain sees it — no extra library needed.
- **A game that moves to its own top-level domain** (e.g. `kattrap.com` instead of `kattrap.kpground.com`) can't use that cookie — cookies don't cross top-level domains, this is a hard browser boundary, not a config limit. For that case, add an OAuth2 identity-provider layer on top of the same accounts: Doorkeeper (Rails) or django-oauth-toolkit (Django). The game does a "Sign in with kpground" redirect flow, same pattern as "Sign in with Google." This also gives a "which games am I logged into" list essentially for free, since OAuth2 providers already track grants (user, client, last used) — that's just surfacing existing data.
- Only build the OAuth2 layer for a game once it actually gets its own domain. Until then the plain cookie approach is simpler and sufficient.

## Multiplayer (build when a specific game needs it)

Action Cable (Rails) or Django Channels (Django) + Redis — standard real-time approach for these frameworks, nothing new to learn.

## What's live vs. deferred

- **Live now**: Cloudflare account created (personal Google account), `kpground.com` added on Cloudflare's Free plan, nameservers switched at Squarespace to `cris.ns.cloudflare.com` / `jean.ns.cloudflare.com` (submitted 2026-07-28, DNSSEC disabled to allow the switch, no email was configured on the domain so nothing was at risk). Repo pushed to `github.com/sprelewicz2441/kpground`. Render account created (GitHub-connected, only `kpground` repo authorized, not all repos), landing page deployed as a Render Static Site (name `kpground`, publish directory `.`, no build command) — live at `https://kpground.onrender.com`. Custom domain wired up: Cloudflare DNS has `kpground.com` and `www.kpground.com` as CNAMEs to `kpground.onrender.com`, set to **DNS only** (grey cloud, not proxied) — Cloudflare's proxy was left off to avoid SSL-mode complications with Render's own cert issuance; both domains verified in Render, `www` redirects to the root domain. `kpground.com`'s SSL certificate finished issuing shortly after `www`'s — confirmed live at `https://kpground.com` on 2026-07-28. Old Squarespace-era `_domainconnect` CNAME record still sits in Cloudflare DNS, harmless leftover, safe to delete whenever.
- **Explicitly deferred (not forgotten)**: **Neon signup/setup** — saved for later on purpose, do this when a shared database is actually needed (landing page or a game auth flow), not before.
- **Deferred until actually needed**: landing page backend + auth, each game's Rails/Django backend, OAuth2 layer (only for a game that gets its own domain), multiplayer (only for a game that needs it).

Don't build ahead of this list — the static landing page and any purely-static game don't need a framework, a database, or auth until there's an actual feature that requires one.
