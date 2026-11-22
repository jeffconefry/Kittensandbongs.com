# kittensandbongs.com

One cute kitten and one bong, every day. Photos come from the Pexels API, there's a
double opt-in mailing list for a daily email, and a tip-jar link. Humor site, 21+.

Built on the same base as the new danielleroney.com (Astro on Cloudflare Workers,
deployed by Workers Builds from GitHub), minus the parts this site doesn't need.

## How it works

```
 Cron (hourly, :07) ──► Worker ──► before 7 a.m. ET: nothing
                          │        after: pick today's kitten + bong (once) ──► D1
                          │               email active subscribers not yet sent today
                          ▼
 Visitors ──► Worker (Astro) ──► reads today's pair / archive from D1
                                  photos load straight from images.pexels.com
 Signup form ──► /api/subscribe ──► Turnstile check ──► confirm email ──► /confirm ──► active
 Daily email ──► one-click unsubscribe (List-Unsubscribe header) ──► /api/unsubscribe
```

Everything is one Worker: `src/worker.ts` passes requests to Astro and runs the cron job.
The job is safe to repeat: the `days` row stops a second pick, and each subscriber's
`last_sent_day` stops a second email. If a run fails, the next hour picks up where it stopped.

### Compared with the danielleroney.com playbook

| Playbook piece | Here | Why |
|---|---|---|
| Astro 7.4 + `@astrojs/cloudflare` 14, Workers Builds on push to `main` | Same | |
| Committed `wrangler.jsonc` (audit T4), `check` script (T6) | Same, from day one | |
| `_headers`, middleware headers, CSP via `security.csp`, `robots.txt` gate on `PUBLIC_SITE_INDEXABLE` | Same | |
| Contact-form protections (origin check, size cap, honeypot, fill-time window, Turnstile) | Applied to the signup form, with the audit fixes: Turnstile required in prod, Origin header required, no message bodies logged | |
| Node adapter in dev | Dropped | No Keystatic; dev runs in workerd so the D1 binding works locally |
| Keystatic, R2, NAS → rclone sync, Image Transformations, media-check | Dropped | No editors, and photos come from Pexels' CDN |
| Self-hosted fonts | Same idea (`@fontsource/anton`, `@fontsource/space-mono`) | |
| New: D1 database, cron trigger, email sending | | The daily job and the list |
| CSP styles | Hashes only, **no** `'unsafe-inline'` | So no `style=""` attributes anywhere in pages |

## First-time setup

Build and push from a working copy with git and node_modules (for example
`S:\CLAUDE\KITTENSANDBONGS\site`), not the Synology Drive mirror.

1. **Repo.** Clone `jeffconefry/Kittensandbongs.com` and run `npm install`.
2. **Database.**
   ```sh
   npx wrangler d1 create kittensandbongs
   ```
   Paste the printed `database_id` into `wrangler.jsonc`, commit, then:
   ```sh
   npm run db:migrate:remote
   ```
3. **Pexels key.** Free at <https://www.pexels.com/api/>. The site must show a link to Pexels
   (it's in the footer and on every photo credit).
4. **Turnstile.** Dashboard → Turnstile → add a widget for `kittensandbongs.com` and
   `kittensandbongs-com.jeffconefry.workers.dev`. You get a site key (public) and a secret.
5. **Worker.** Workers & Pages → Create → import the repo. Build command `npm run build`,
   deploy command `npx wrangler deploy`, production branch `main`.
   - Build variables: `PUBLIC_TURNSTILE_SITE_KEY`, `PUBLIC_TIP_URL` (optional). Leave
     `PUBLIC_SITE_INDEXABLE` unset until launch.
   - Secrets (Settings → Variables and Secrets, type Secret): `PEXELS_API_KEY`,
     `TURNSTILE_SECRET_KEY`, `ADMIN_TOKEN` (long random string), and later `EMAIL_API_KEY`.
   - The first deploy also creates a `SESSION` KV namespace automatically (the Astro
     adapter asks for it). It's unused and harmless.
6. **Test with email off.** `EMAIL_PROVIDER` starts as `log`: emails are printed to the
   Worker logs, not sent. Publish today's pair by hand and check the site:
   ```sh
   curl -X POST -H "Authorization: Bearer $ADMIN_TOKEN" https://kittensandbongs-com.jeffconefry.workers.dev/api/admin/run
   ```
7. **Email (do this carefully).** Pick a sending service whose acceptable use policy allows
   this content; many mainstream ones ban cannabis-related email. Then:
   - verify the sending domain with the provider (it gives SPF/DKIM DNS records to add in Cloudflare) and add a DMARC record;
   - set `EMAIL_PROVIDER` (`resend` is built in; other providers are a small case in `src/lib/email.ts`), `EMAIL_FROM`, and `EMAIL_API_KEY`;
   - set `POSTAL_ADDRESS` (a PO box is fine). US anti-spam law requires it in every email, so don't send real emails without it.
8. **Launch.** Add the custom domain to the Worker (Settings → Domains & Routes). Set
   `PUBLIC_SITE_INDEXABLE=true` and rebuild if you want search engines to list it.

### Also from day one (open items on danielleroney.com)
- Security → WAF → Rate limiting rule: `/api/subscribe`, 5 requests per minute per IP, block.
- GitHub → Settings → Code security: turn on Dependabot alerts (`.github/dependabot.yml` is already in the repo).
- Keep the tip jar on its own account, separate from studio payments. Payment processors restrict drug paraphernalia, and an automated review could freeze it.

## Day to day

Nothing to do: the site updates itself at 7 a.m. Eastern (`SEND_HOUR` in `wrangler.jsonc`).

**Hide a bad photo** (wrong subject, anything you don't want up):
```sh
npx wrangler d1 execute kittensandbongs --remote --command "UPDATE photos SET blocked = 1 WHERE pexels_id = 12345"
```
The Pexels id is in the photo's Pexels link. A day with a blocked photo disappears from the
site and the archive. To pick a replacement for **today**:
```sh
npx wrangler d1 execute kittensandbongs --remote --command "DELETE FROM days WHERE day = '2026-11-15'"
curl -X POST -H "Authorization: Bearer $ADMIN_TOKEN" https://kittensandbongs.com/api/admin/run
```
People who already got today's email won't get a second one.

**Subscriber count:**
```sh
npx wrangler d1 execute kittensandbongs --remote --command "SELECT status, COUNT(*) FROM subscribers GROUP BY status"
```

**Delete someone's address entirely** (if they ask):
```sh
npx wrangler d1 execute kittensandbongs --remote --command "DELETE FROM subscribers WHERE email = 'them@example.com'"
```

## Local development

```sh
cp .dev.vars.example .dev.vars   # add PEXELS_API_KEY and an ADMIN_TOKEN
npm run db:migrate:local
npm run dev                      # http://localhost:4321
```
In dev, signup works without Turnstile and emails are only logged (`astro dev logs`).
`curl -X POST -H "Authorization: Bearer <ADMIN_TOKEN>" localhost:4321/api/admin/run` runs the daily job.

After changing `wrangler.jsonc`, run `npm run types`. `npm run check` type-checks everything.

## Limits worth knowing

- **Pexels:** free key, 200 requests/hour, 20,000/month. A daily pick uses a handful.
- **Workers Free plan:** each cron run sends at most 500 emails (5 batches of 100); a bigger
  list just finishes over the next hours. Raise `MAX_BATCHES_PER_RUN` in `src/lib/daily.ts` on a paid plan.
- **Bong photos:** stock libraries have far fewer bongs than kittens. Bong picks only accept
  photos whose description mentions a bong or water pipe, and never repeat. If that pool runs
  dry, the job fails loudly in the logs rather than posting something off-topic. Add search
  terms in `src/lib/pexels.ts` when that happens.

## Files

```
wrangler.jsonc            Worker name, cron, D1 binding, non-secret settings
astro.config.mjs          adapter, CSP, checkOrigin off (one-click unsubscribe)
migrations/0001_init.sql  photos, days, subscribers
src/worker.ts             entry: Astro fetch + cron
src/lib/daily.ts          the daily job (pick + send, resumable)
src/lib/pexels.ts         photo search and filtering
src/lib/email.ts          provider adapter (log, resend)
src/lib/templates.ts      daily + confirm emails (HTML and plain text)
src/lib/guard.ts          origin, form size, Turnstile, admin token checks
src/pages/                home, archive, day/[date], confirm/unsubscribe flow, privacy, 404, api/*
src/layouts/Base.astro    header, footer, 21+ screen
public/_headers           security headers for static files
```
