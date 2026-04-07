---
title: "Defending against npm supply chain attacks with min-release-age on App Service Linux"
author_name: "Takeharu Oshida"
tags:
    - Nodejs
    - Deployment
    - Oryx
    - NPM
    - Zip Deploy
categories:
    - Azure App Service on Linux
    - Nodejs
    - Deployment
header:
    teaser: /assets/images/nodelinux.png
toc: true
toc_sticky: true
date: 2026-04-07 06:00:00
---

On March 31, 2026, the popular npm package **axios** was compromised through a supply chain attack ([Post Mortem](https://github.com/axios/axios/issues/10636)). Malicious versions `1.14.1` and `0.30.4` were published to the npm registry and remained available for approximately 3 hours before being removed.

In this post, we will explore how to use npm's `min-release-age` configuration in `.npmrc` to protect your App Service Linux deployments from this type of attack by ensuring that only packages published more than a given number of days ago are installed.

# Overview

## What is min-release-age?

`min-release-age` is a configuration option introduced in **npm 11.10.0** (released February 11, 2026). When set, npm will only install package versions that were published more than the specified number of days ago.

```ini
# .npmrc
min-release-age=7
```

With this setting, if a malicious version of a package is published today, it will not be eligible for installation for 7 days — giving the community time to detect and remove it.

## How would this have helped with the axios attack?

The axios attack timeline was:

| Time (UTC) | Event |
|---|---|
| March 31, 00:21 | Malicious `axios@1.14.1` published |
| March 31, ~01:00 | Malicious `axios@0.30.4` published |
| March 31, ~01:00 | Community members detect the compromise |
| March 31, 03:15 | Malicious versions removed from npm |

The malicious versions were live for about **3 hours**. If `min-release-age=3` had been set:

- `axios@1.14.1` (published less than 3 days ago) would **not** be eligible for installation
- The safe `axios@1.14.0` would have been selected automatically
- **The attack would have been completely avoided**

# Prerequisites

- Azure App Service Linux with **Node.js 24 LTS** runtime
- Application setting `SCM_DO_BUILD_DURING_DEPLOYMENT=true` (to enable Oryx remote build)
- A Node.js project with a `package.json` file

# Checking the npm version in Oryx

Before configuring `min-release-age`, it is important to verify which npm version Oryx uses for your build. At the time of writing, Oryx ships with **npm 11.6.2** for Node.js 24 — which is **below the 11.10.0 requirement**.

You can check the npm version in the Oryx build logs. Deploy your project and look for the following lines:

```
Using Node version:
v24.13.0

Using Npm version:
11.6.2
```

Build logs can be viewed from the **Deployment Center** in the Azure Portal, or directly via SSH/Kudu at `/home/LogFiles/kudu/deployment/`.

> Since `min-release-age` requires npm 11.10.0 or higher, we need to upgrade npm as part of the build process. We can use the `PRE_BUILD_COMMAND` app setting to run a script before `npm install`. See the [Oryx configuration documentation](https://github.com/microsoft/Oryx/blob/main/doc/configuration.md) for details on `PRE_BUILD_COMMAND` and other build hooks.

# Setting up min-release-age with Oryx

## Step 1: Add .npmrc to your project

Create a `.npmrc` file in the root of your project:

```ini
min-release-age=7
```

This tells npm to only install versions that were published **7 or more days ago**.

## Step 2: Create a pre-build script

Since Oryx currently ships npm 11.6.2 (which does not support `min-release-age`), we need a pre-build script to upgrade npm to a version that supports it.

Create `scripts/deploy/pre-build.sh`:

```bash
#!/bin/bash
# pre-build.sh
# Check npm version before Oryx build and upgrade to 11.10.0
# if the current version does not support min-release-age.

NPM_VER=$(npm --version)
NPM_MAJOR=$(echo "$NPM_VER" | cut -d. -f1)
NPM_MINOR=$(echo "$NPM_VER" | cut -d. -f2)

if [ "$NPM_MAJOR" -lt 11 ] || ([ "$NPM_MAJOR" -eq 11 ] && [ "$NPM_MINOR" -lt 10 ]); then
  echo ">>> npm $NPM_VER does not support min-release-age (requires >= 11.10.0)"
  echo ">>> Installing npm@11.10.0..."
  npm install -g npm@11.10.0
  echo ">>> npm updated to $(npm --version)"
else
  echo ">>> npm $NPM_VER already supports min-release-age"
fi
```

> **Note:** We specify `npm@11.10.0` (a fixed version) instead of `npm@latest` to avoid introducing risks from a potentially compromised latest npm release. Once Oryx ships npm 11.10.0 or higher, this script will automatically skip the upgrade.

> **Important:** This pre-build script (`scripts/deploy/pre-build.sh`) must be included in your deployment package (e.g., the ZIP file). Oryx executes `PRE_BUILD_COMMAND` from the deployed source directory, so the script will not be found if it is excluded from the deployment artifact.

## Step 3: Configure the app setting

Set the `PRE_BUILD_COMMAND` app setting to point to your script:

```bash
az webapp config appsettings set \
  --resource-group <your-resource-group> \
  --name <your-app-name> \
  --settings PRE_BUILD_COMMAND="./scripts/deploy/pre-build.sh"
```

## Step 4: Deploy

Deploy your application using ZIP deploy:

```bash
az webapp deploy \
  --resource-group <your-resource-group> \
  --name <your-app-name> \
  --src-path <your-zip-file> \
  --type zip
```

# Verification

We tested this configuration using a Next.js project with the dependency `"next": "^16.2.0"`. At the time of testing (April 7, 2026), the following stable versions were available:

| Version | Published | Age (days) |
|---|---|---|
| 16.2.2 | 2026-04-01 | 6 |
| 16.2.1 | 2026-03-20 | 18 |
| 16.2.0 | 2026-03-18 | 20 |
| 16.1.7 | 2026-03-16 | 22 |

## Test 1: Without min-release-age (baseline)

With no `.npmrc` configuration, the latest matching version `16.2.2` was installed:

```
Using Npm version:
11.6.2

Running 'npm install'...
added 22 packages, and audited 23 packages in 19s

▲ Next.js 16.2.2 (Turbopack)
```

![Oryx build log without min-release-age](/media/2026/04/npm-install-without-min-release-age.png)

## Test 2: With min-release-age=7

With `.npmrc` containing `min-release-age=7` and the pre-build script upgrading npm:

```
Executing pre-build command...
npm warn Unknown project config "min-release-age". This will stop working in the next major version of npm.
>>> npm 11.6.2 does not support min-release-age (requires >= 11.10.0)
>>> Installing npm@11.10.0...
removed 40 packages, and changed 83 packages in 12s
>>> npm updated to 11.10.0

Using Npm version:
11.10.0

Running 'npm install'...
added 22 packages, and audited 23 packages in 20s

▲ Next.js 16.2.1 (Turbopack)
```

![Oryx build log with min-release-age=7](/media/2026/04/npm-install-with-min-release-age.png)

**Result:** Version `16.2.2` (published 6 days ago) was excluded because it did not meet the 7-day minimum. Instead, `16.2.1` (published 18 days ago) was installed.

> The warning `npm warn Unknown project config "min-release-age"` is generated by the **older npm 11.6.2** when it reads the `.npmrc` during the pre-build step. After npm is upgraded to 11.10.0, the setting is properly recognized during `npm install`.

# Selecting a min-release-age value

There is no officially recommended value in the npm documentation or RFCs. The appropriate value depends on your project's requirements and risk tolerance. The following table provides a reference for the trade-offs of each setting:

| Value | Characteristics | Trade-off |
|---|---|---|
| `1` | Minimal delay, avoids only same-day attacks | Most attacks are detected within hours, but not all |
| `3` | Suitable for fast-moving projects | Would have prevented the axios attack (3-hour window) |
| `7` | Allows a full week for community detection | Security patches may be delayed up to 1 week |
| `14` | Conservative operation | Security patches may be delayed up to 2 weeks |
| `30` | Very conservative | Significant delay in receiving updates, including security fixes |

For reference, pnpm provides a similar feature `minReleaseAge` configured in minutes ([pnpm Supply Chain Security](https://pnpm.io/supply-chain-security)), and Dependabot also has a comparable [minimumReleaseAge](https://docs.github.com/en/code-security/dependabot/dependabot-version-updates/configuration-options-for-the-dependabot.yml-file#minimum-release-age) setting. Consider these tools' defaults and recommendations when choosing a value for your project.

# Important considerations

## min-release-age vs --before

npm also provides a `--before` flag that accepts an absolute date. These two options **cannot be used together** — npm will return an error if both are set.

This is explicitly documented in the npm config reference: [min-release-age](https://docs.npmjs.com/cli/v11/using-npm/config#min-release-age) states `This config cannot be used with: before`, and [before](https://docs.npmjs.com/cli/v11/using-npm/config#before) states `This config cannot be used with: min-release-age`. Internally, `min-release-age` is converted to a `before` date value, making them mutually exclusive ([PR #8965](https://github.com/npm/cli/pull/8965)).

| | `min-release-age` | `--before` |
|---|---|---|
| Type | Relative (number of days) | Absolute (specific date) |
| Requires updating | No | Yes (date becomes stale) |
| npm version | 11.10.0+ | 7.0.0+ |
| Configuration | `.npmrc` | `.npmrc` or environment variable |

`min-release-age` is preferred because it is a **set-and-forget** configuration that does not require updating over time.

## When Oryx upgrades its npm version

Once Oryx ships with npm 11.10.0 or higher, the pre-build script will detect this and skip the upgrade:

```
>>> npm 11.10.0 already supports min-release-age
```

At that point, you only need the `.npmrc` file — the pre-build script becomes a no-op but can remain as a safety net.

## Lock files and Oryx build behavior

Oryx runs **`npm install`** (not `npm ci`) when building Node.js projects ([Oryx Node.js build documentation](https://github.com/microsoft/Oryx/blob/main/doc/runtimes/nodejs.md)).

While `npm ci` strictly reproduces the lock file contents, `npm install` may resolve new versions based on the semver ranges in `package.json` even when a lock file is present. Therefore, **committing `package-lock.json` alone does not provide complete protection in Oryx remote builds**.

`min-release-age` is particularly effective as an additional defense layer against this `npm install` behavior:

- Running `npm install` without a lock file (e.g., first-time builds)
- A lock file exists but Oryx's `npm install` resolves newer versions
- Deliberately running `npm update` to get newer versions

> If you want to use `npm ci` with Oryx, you can set the app setting `CUSTOM_BUILD_COMMAND="npm ci"`. However, note that this skips Oryx's default `npm install` and `npm run build` execution, so you will need to manage the entire build command yourself.

## This does not replace other security practices

`min-release-age` is one layer in a defense-in-depth strategy. Continue to:

- Run `npm audit` regularly to check for known vulnerabilities
- Use `npm audit signatures` to verify package provenance
- Pin critical dependencies to exact versions when appropriate
- Review the [npm security best practices](https://docs.npmjs.com/packages-and-modules/securing-your-code)

# Additional References

- [axios npm supply chain compromise - Post Mortem (GitHub Issue #10636)](https://github.com/axios/axios/issues/10636)
- [npm config: min-release-age](https://docs.npmjs.com/cli/v11/using-npm/config#min-release-age)
- [npm CLI: add min-release-age (PR #8965)](https://github.com/npm/cli/pull/8965)
- [Generating provenance statements - npm Docs](https://docs.npmjs.com/generating-provenance-statements)
- [Oryx - Build system for App Service](https://github.com/microsoft/Oryx)
- [Troubleshooting Node.js deployments on App Service Linux](https://azureossd.github.io/2023/02/09/troubleshooting-nodejs-deployments-on-appservice-linux/index.html)
