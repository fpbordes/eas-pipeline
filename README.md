# eas-pipeline

One reusable GitHub Actions workflow for every Expo app built with
[EAS](https://docs.expo.dev/eas/) (Expo Application Services): publish an over-the-air update,
build the store binaries, or build and submit them to the App Store and Google Play. Apps call it
instead of carrying a near-identical workflow each.

It lives in a public repository on purpose: GitHub lets a private repository call a reusable
workflow only from repositories of the same owner, and the apps calling this one belong to
several owners.

## How an app publishes and updates itself

| Event in the app repository | Call with | What happens |
|---|---|---|
| Push to `main` | `mode: update` | `eas update` publishes the JavaScript of the commit to the profile's channel. Phones pick it up at their next start. No store review. |
| Tag `v<version>` | `mode: release` | `eas build` for iOS and Android in parallel, then `eas submit` of each binary to its store with the submit profile of `eas.json`. |
| By hand (`workflow_dispatch`) | `mode: build` | Binaries only, to test by hand; `latest.json` names both builds. |

A release is therefore one gesture: push a tag. A native change (a new Expo SDK, a new permission,
a new native module) needs a release; everything else ships as an update.

## Calling it

Add `.github/workflows/eas.yml` to the app. One job per brand the app ships: each names its own
profile of `eas.json` and maps its own brand-prefixed secrets onto the generic names this workflow
declares (the `secrets` context cannot be indexed by a variable, so the mapping happens here).

```yaml
name: EAS

on:
  push:
    branches: [main]
    tags: ['v*']
  workflow_dispatch:
    inputs:
      mode:
        type: choice
        options: [update, build, release]
        default: build

jobs:
  wesmart:
    uses: fpbordes/eas-pipeline/.github/workflows/eas.yml@main
    with:
      app_name: wesmart
      profile: production
      mode: ${{ startsWith(github.ref, 'refs/tags/v') && 'release' || inputs.mode || 'update' }}
      asc_api_key_path: ./.secrets/AuthKey_XXXXXXXXXX.p8   # as eas.json → submit.<profile>.ios.ascApiKeyPath
    secrets:
      EAS_TOKEN: ${{ secrets.WESMART_EAS_TOKEN }}
      EXPO_APPLE_TEAM_ID: ${{ secrets.WESMART_EXPO_APPLE_TEAM_ID }}
      ASC_API_KEY_P8: ${{ secrets.WESMART_ASC_API_KEY_P8 }}
      GOOGLE_SERVICE_ACCOUNT_KEY: ${{ secrets.WESMART_GOOGLE_SERVICE_ACCOUNT_KEY }}
```

### Inputs

| Input | Default | Meaning |
|---|---|---|
| `app_name` | required | App or brand identifier; names artifacts and log lines |
| `mode` | `update` | `update`, `build` or `release` |
| `profile` | `production` | Build profile of the caller's `eas.json` (one per brand when the app ships several) |
| `submit_profile` | the build profile | Submit profile of `eas.json`, for `release` |
| `channel` | the profile's name | EAS Update channel, for `update` |
| `asc_api_key_path` | — | Where `eas.json` expects the App Store Connect API key file; written from `ASC_API_KEY_P8` |
| `google_service_account_key_path` | — | Where `eas.json` expects the Google Play service-account JSON; written from `GOOGLE_SERVICE_ACCOUNT_KEY` |
| `node_version` | `20` | Node.js version |
| `dry_run` | `false` | Print the `eas` commands instead of running them |

### Secrets

| Secret | Used for |
|---|---|
| `EAS_TOKEN` | Every non-interactive `eas` command. **Its absence switches the whole run to dry run**, so a caller can be wired before the app's EAS project or store accounts exist. |
| `EXPO_APPLE_TEAM_ID` | iOS signing |
| `EXPO_APPLE_APP_SPECIFIC_PASSWORD` | Submission with an Apple ID (submit profile with `appleId`) |
| `ASC_API_KEY_P8` | Submission with an App Store Connect API key (submit profile with `ascApiKeyPath`) |
| `GOOGLE_SERVICE_ACCOUNT_KEY` | Android credentials and Google Play submission |

Declare them in the calling repository, prefixed with the brand in upper case
(`WESMART_EAS_TOKEN`, `BELVUS_EAS_TOKEN`, `EKWO_EAS_TOKEN`…), and map them in the call.

## Why `latest.json` never races

The two build jobs are independent. Each writes a result file of its own and uploads it as an
artifact; a last job, gated on both through `needs:`, downloads every entry and writes
`latest.json` once. No job ever reads and rewrites a manifest another job may be writing at the
same instant — the race that once lost a platform entry in a desktop release pipeline.

## Scope

The workflow builds, updates and submits. It never edits `eas.json` or `app.json`, never creates a
secret, never touches the store listing (name, screenshots, review notes): those stay in the app's
repository and in its store console.

## License

MIT.
