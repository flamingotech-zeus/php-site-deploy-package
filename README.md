# PHP Site Deploy Package

[English](README.md) | [简体中文](README.zh-CN.md)

> Package a static front end plus native PHP endpoints into an uploadable bundle, and run it through a real PHP engine locally before it ever reaches the server.

## What this is

> **Note on de-identification.** This README is written generically. The skill file itself may still reference internal domains, host names or local absolute paths. Run a scan pass (see `opensource-preflight-audit`) and replace them before publishing.

This is not just a zipping step. The failure this skill is built around is a specific one: everything works locally because the static files are served by a development server, then the upload reveals that the PHP half never ran at all.

The fix is to make the package runnable before it ships. Build the bundle, execute it locally against a real PHP engine, and only then transfer it.

## Why we open-sourced it

We deploy this kind of site regularly — a static front end with a handful of PHP endpoints behind it. The local-passes, server-fails gap cost us a few evenings before we automated the check away.

It is a deployment procedure, not a product. Nothing of ours is inside it.

## What it does

- **Package, then run it locally.** The bundle is executed against a real PHP engine before upload, so the gap between local and server closes at the point where it is cheap to close.
- **An incremental publisher.** After the site is live, subsequent updates sync only what changed instead of re-uploading everything.
- **A full setup checklist** covering subdomain creation, DNS records, certificate issuance and file transfer, in the order they need to happen.
- **Verification after upload** rather than assuming a successful transfer means a working site.

## Requirements

Windows · PHP · Python 3

## Installation

```bash
git clone https://github.com/<your-username>/php-site-deploy-package.git
```

Then place the folder into your agent skills directory:

| OS | Path |
|---|---|
| Windows | `C:\Users\<you>\.workbuddy\skills\php-site-deploy-package\` |
| macOS / Linux | `~/.workbuddy/skills/php-site-deploy-package/` |

There is no build step. Restart the session and it takes effect.

## How to use

Ask for the deployment:

- "Package this and check it before uploading"
- "Deploy these changes to the live site"
- "Set up the subdomain from scratch"

The skill builds, verifies locally, uploads, then confirms on the server.

## Files

```
└── SKILL.md
```

## Notes and limits

1. **A local dev server does not run PHP.** Serving static files successfully proves nothing about the PHP half. This is the entire reason the local execution step exists.
2. **Incremental publishing needs a clean baseline.** If the live site has drifted from the packaged version, do a full publish first.
3. **Certificates and DNS take time to propagate.** The checklist orders the steps so you are not waiting on a record that should have been created earlier.
4. **Back up before publishing.** The increment publisher reduces the blast radius; it does not remove the need for a copy.

## License

MIT
