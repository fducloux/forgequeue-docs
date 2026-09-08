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

**iOS** needs a distribution certificate and a provisioning profile.
Two ways to provide them:

- **Upload your own** — export a `.p12` from Keychain Access (with its
  password) and download a matching `.mobileprovision` from Apple
  Developer, then upload both on the repo's Signing page.
- **Let forgeQueue generate them for you** — save a Team API key from App
  Store Connect (Users and Access → Integrations → Team Keys: download the
  `.p8`, note the Key ID and Issuer ID), then click **Generate**. forgeQueue
  registers the bundle ID, requests a certificate, and creates a
  provisioning profile automatically — no manual export needed.

If you manage multiple apps under one team, you can save signing once as
your **team default** and only override it per-repo for apps that need
their own certificate.

**Android** needs your signing keystore, its password, and your key alias
+ password — uploaded the same way on the repo's Android Signing page.

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
