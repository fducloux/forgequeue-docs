# forgeQueue docs

Push to a branch, get a signed iOS or Android build back — no Mac
required. This repo is documentation only: how to connect a repo to
[forgeQueue](https://forgequeue.8rec.com) and trigger your first build.
It contains no application source code.

If you're using Claude Code in a project you want to connect to
forgeQueue, point it at `.claude/skills/forgequeue/` in this repo (or copy
that folder into your own project's `.claude/skills/`) and it'll pick up
the integration guide automatically.

## 1. Sign in and link your repo

1. Go to **[forgeQueue.8rec.com](https://forgequeue.8rec.com)** and sign in
   with GitHub.
2. Install the forgeQueue GitHub App on your repository (or your whole
   organization, if you'll be adding more repos later).
3. From the dashboard, add the repo. forgeQueue creates a dedicated CI
   branch for it (`forgeQueue/ci-release` by default) — this is the branch
   you'll push to trigger builds.

## 2. Tell forgeQueue what to build

On the repo's settings page, turn on iOS and/or Android — you can build
either or both from the same repo. If your project lives in a subfolder
(common for monorepos with separate `/ios` and `/android` folders), set
that path too.

## 3. Set up code signing

**iOS** needs a certificate and, separately, a provisioning profile — a
certificate isn't tied to any one app, but a profile is:

- **Certificate** — set once on your **team's** Signing page. Every app on
  the team can share it. Upload a `.p12` (+ its password) from Keychain
  Access, or save a Team API key from App Store Connect (Users and Access
  → Integrations → Team Keys: the `.p8`, Key ID, Issuer ID) and click
  **Generate** instead.
- **Bundle ID + provisioning profile** — set on **each app's own** Signing
  page, every time, even when several apps share the team's certificate.
  Upload a `.mobileprovision`, or generate one there once the certificate
  exists.

**Android** needs your signing keystore, its password, and your key alias
+ password — one per app, uploaded on that app's Android Signing page (no
auto-generate for Android).

## 4. Trigger a build

Push a commit to the repo's CI branch:

```bash
git push origin forgeQueue/ci-release
```

That's it — no config file, no CI YAML to write. forgeQueue picks up the
push, queues a build, and a worker claims and runs it.

## 5. Watch it build

Open the repo on your **dashboard** to see:

- Live build log, streamed as it runs
- Queue position and an estimated time to completion
- Credits charged for the build
- If something fails, an automatic diagnosis of what went wrong

## 6. Get your build

When it finishes successfully, the signed `.ipa` or `.aab` is pushed back
into your own repo, on a dedicated branch: **`forgeQueue/builds`**, under
`forgequeue-builds/<build-id>/`. It's never pushed onto your CI branch, so
grabbing a build never triggers another one.

## Checking build status from a script

For polling outside the dashboard — a CI job, a bot, your own tooling —
generate a per-team API key on **Account → Team → API access** (shown
once at creation; losing it means rotating, not recovering it):

```bash
curl https://forgequeue.8rec.com/api/public/builds \
  -H "Authorization: Bearer fq_live_..."

curl https://forgequeue.8rec.com/api/public/builds/<build-id> \
  -H "Authorization: Bearer fq_live_..."
```

The list is newest-first and filterable with `?repoId=`, `?platform=`,
`?status=`; page through it with `?limit=` (default 20, max 100) and
`?cursor=` (from the previous response's `nextCursor`). List rows skip
the log to stay light — fetch the single build for that.

The same key also manages env vars and gitignored config files
end-to-end:

```bash
curl -X PUT https://forgequeue.8rec.com/api/public/repos/<repo-id>/env \
  -H "Authorization: Bearer fq_live_..." -H "Content-Type: application/json" \
  -d '{"dotenv": "NEXT_PUBLIC_API_URL=https://api.example.com"}'

curl -X PUT https://forgequeue.8rec.com/api/public/repos/<repo-id>/env/files \
  -H "Authorization: Bearer fq_live_..." \
  -F "targetPath=ios/App/App/GoogleService-Info.plist" \
  -F "file=@./GoogleService-Info.plist"
```

## Gitignored config files (GoogleService-Info.plist, google-services.json, ...)

For files a build needs at a specific path but that are correctly kept
out of git because they're credential-like. On the repo's **Env** page,
a second form lets you upload one and give the path it needs to land at,
relative to the repo root — written into the checkout before any build
step runs.

## Billing

Builds are charged in credits based on build time, whether they succeed or
fail (like any CI provider — you're paying for the compute either way).
Before your first build on a repo, forgeQueue shows an estimated cost; once
you have a build history, the estimate is based on your own average. If
your balance runs low, a build is held rather than failing outright — top
up and it'll run automatically.

## Questions or issues

Use the **Issues** tab on your dashboard to report a bug, dispute a charge,
or request a feature — it goes straight to the team behind forgeQueue.
