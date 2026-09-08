---
name: forgequeue
description: Use whenever a project needs iOS or Android CI builds — a signed .ipa/.aab from a push, no Mac required — via forgeQueue (forgequeue.8rec.com). Also use when onboarding a repo onto forgeQueue, debugging why a push didn't trigger a build, or setting up iOS/Android code signing on it.
---

# forgeQueue

Push-to-build CI for iOS and Android: push to a configured branch on a
GitHub repo linked to forgeQueue, it builds and signs the app, and pushes
the compiled artifact back into the same repo.

- Dashboard: `https://forgequeue.8rec.com`
- There is no REST API to integrate against. The entire integration
  surface is **`git push`** to a specific branch, plus the web dashboard
  for setup and status. If a task here seems to call for an API key or
  SDK, that's a sign of confusing forgeQueue with a different service —
  stop and re-check.

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

**iOS** needs a distribution certificate and a provisioning profile — two
ways to provide them, both on the repo's Signing page:

1. **Upload your own** — a `.p12` (+ password) exported from Keychain
   Access, and a matching `.mobileprovision`.
2. **Generate automatically** — save a Team API key from App Store
   Connect (Users and Access → Integrations → Team Keys: the `.p8`, Key
   ID, and Issuer ID) first, then click **Generate**. forgeQueue registers
   the bundle ID, requests a certificate, and creates the profile for
   you.

Get the bundle ID right before generating — a provisioning profile is
locked to one App ID, and a mismatch against the Xcode project's real
`PRODUCT_BUNDLE_IDENTIFIER` fails every build at the codesign step. Check
the actual value in the project rather than trusting a bundle ID recorded
somewhere else.

Signing can be set once as a **team default** and shared across repos, or
overridden per repo for an app that needs its own certificate.

**Android** takes a keystore, its password, and a key alias + password,
uploaded the same way — no auto-generate flow for Android.

## Checking build status

Dashboard only (`forgequeue.8rec.com/dashboard`) — there's no
API-key-authenticated status endpoint for external tools to poll. It
shows the live build log, queue position + ETA, credits charged, and an
automatic error diagnosis on failure.

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
