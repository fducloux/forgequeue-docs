---
name: forgequeue
description: Use whenever a project needs iOS or Android CI builds — a signed .ipa/.aab from a push, no Mac required — via forgeQueue (forgequeue.8rec.com). Also use when onboarding a repo onto forgeQueue, debugging why a push didn't trigger a build, or setting up iOS/Android code signing on it.
---

# forgeQueue

Push-to-build CI for iOS and Android: push to a configured branch on a
GitHub repo linked to forgeQueue, it builds and signs the app, and pushes
the compiled artifact back into the same repo.

- Dashboard: `https://forgequeue.8rec.com`
- Triggering a build is still `git push` only. There's a small read-only
  API for checking build status (see below) using a per-team API key —
  everything else (onboarding, signing, triggering) is dashboard/git only,
  no SDK.

## Onboarding a repo (one-time, via the dashboard — not scriptable)

Done once per repo by whoever owns it, signed in at forgequeue.8rec.com:

1. Sign in with GitHub, install the forgeQueue GitHub App on the repo (or
   org).
2. **Dashboard → Repos → add repo.** This assigns a CI branch — defaults
   to `forgeQueue/ci-release`, but check the dashboard for the actual
   value rather than assuming the default.
3. **Dashboard → Repos → [repo] → Platforms** — turn on iOS and/or
   Android. If the project lives in a subfolder (a monorepo with separate
   `/ios` and `/android` trees), set that path too.
4. **Dashboard → Repos → [repo] → Signing** (Android has its own variant
   of this page) — see below.
5. Confirm the repo has credit to build with (team-shared by default, or
   billed to the owner's personal balance if they've turned that off). No
   credit source, no builds — a push just queues a build in a held state
   instead of running it.

None of this is scriptable — it's an OAuth + browser-session flow. A bot
account can't do it unattended.

## Triggering a build

Push to the repo's configured CI branch. That's the whole trigger:

```bash
git push origin forgeQueue/ci-release
```

A push to any other branch is ignored. One push can produce **up to two
builds** — one per enabled platform — each billed separately.

**Never push forgeQueue's own build output back onto the CI branch** —
that would re-trigger a build on every successful build, an infinite
loop. Build output always lands on a separate, fixed branch instead (see
below) — never touch that assumption.

## Where the build lands

Never on the CI branch. Always on **`forgeQueue/builds`**, under:

```
forgequeue-builds/<build-id>/app.ipa   (iOS)
forgequeue-builds/<build-id>/app.aab   (Android)
```

Each build gets its own folder, so nothing from one run overwrites
another. Nothing pings you when it lands — check the dashboard, or watch
that branch yourself (e.g. a scheduled fetch+diff, or your own workflow
triggered on pushes to `forgeQueue/builds`).

## Signing

**iOS** needs a certificate and, separately, a provisioning profile — a
certificate isn't tied to one app, a profile is:

- **Certificate** — set once, on the **team's** signing page, shared by
  every app on the team (or upload your own there instead).
- **Bundle ID + provisioning profile** — set on *each app's own* signing
  page, every time, even when apps share the team's certificate. Get the
  bundle ID right — a mismatch against the Xcode project's real
  `PRODUCT_BUNDLE_IDENTIFIER` fails every build at the codesign step.

Both can be uploaded manually or generated automatically (needs a Team
API key from App Store Connect saved on the team's signing page first —
Users and Access → Integrations → Team Keys: the `.p8`, Key ID, Issuer
ID).

**Android** takes a keystore, its password, and a key alias + password —
one per app, uploaded manually, no auto-generate flow.

## Checking build status

Dashboard (`forgequeue.8rec.com/dashboard`) has the most detail: live
build log, queue position + ETA, credits charged, automatic error
diagnosis on failure.

For scripted access, generate a per-team key on **Account → Team → API
access** first:

```bash
curl https://forgequeue.8rec.com/api/public/builds \
  -H "Authorization: Bearer fq_live_..."

curl https://forgequeue.8rec.com/api/public/builds/<build-id> \
  -H "Authorization: Bearer fq_live_..."
```

The list is newest-first, filterable with `?repoId=`/`?platform=`/
`?status=`, paged with `?limit=` (max 100) and `?cursor=`. It omits the
log to stay light — fetch the single build for that.

The same key also manages env vars and config files (below) end to end:

```bash
curl -X PUT https://forgequeue.8rec.com/api/public/repos/<repo-id>/env \
  -H "Authorization: Bearer fq_live_..." -H "Content-Type: application/json" \
  -d '{"dotenv": "NEXT_PUBLIC_API_URL=https://api.example.com"}'

curl -X PUT https://forgequeue.8rec.com/api/public/repos/<repo-id>/env/files \
  -H "Authorization: Bearer fq_live_..." \
  -F "targetPath=ios/App/App/GoogleService-Info.plist" \
  -F "file=@./GoogleService-Info.plist"
```

## Billing

Charged by build time, whether the build succeeds or fails — the same
logic every CI provider bills by. The estimate shown before a build is
your own historical average once you have one, a flat default before
that. Insufficient balance holds a build rather than rejecting the push;
it runs automatically once topped up.

## Repo requirements

- **iOS**: a discoverable `.xcworkspace` or `.xcodeproj`. A root
  `package.json` (Capacitor/Cordova) is built and synced first if
  present — set build-time env vars on the repo's Env page rather than
  committing secrets.
- **Android**: a Gradle project buildable with `bundleRelease`.

## Gitignored config files (GoogleService-Info.plist, google-services.json, ...)

For files a build needs at a specific path but that are correctly kept
out of git because they're credential-like. **Dashboard → Repos →
[repo] → Env**, second form on that page — upload the file and give the
path it needs to land at, relative to the repo root (e.g.
`ios/App/App/GoogleService-Info.plist`). Written into the checkout
before any build step runs. Same page as env vars, but a separate
upload per file — vars and files don't overwrite each other.
