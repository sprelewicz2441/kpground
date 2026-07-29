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

- **Live now**: landing page as static HTML/JS, deployable as a Render Static Site. DNS not yet moved to Cloudflare.
- **Deferred until actually needed**: landing page backend + auth, each game's Rails/Django backend, shared Neon database, OAuth2 layer (only for a game that gets its own domain), multiplayer (only for a game that needs it).

Don't build ahead of this list — the static landing page and any purely-static game don't need a framework, a database, or auth until there's an actual feature that requires one.
