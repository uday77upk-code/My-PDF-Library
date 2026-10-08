# Build Android APK with GitHub Actions — Setup Guide

Project type: **Android Native (Kotlin/Java, Gradle)** · Branch: **main**

## What this workflow does

| Event | Result |
|---|---|
| Push to `main` | Builds a **debug APK**, downloadable from the run's **Artifacts** |
| Publish a GitHub Release | Builds a **signed release APK** and attaches it to the Release (falls back to the debug APK if you haven't set up signing) |
| Manual run (Actions tab → Run workflow) | Same as a push |

---

## Step 1 — Add the workflow to your repo

Copy the folder from this zip into the **root** of your project:

```
your-project/
├── .github/
│   └── workflows/
│       └── android-build.yml   <-- this file
├── app/
├── gradlew
└── build.gradle(.kts)
```

## Step 2 — Check your project

1. `gradlew` and the `gradle/wrapper/` folder must be committed to Git.
2. Your app module must be named `app` (the default). If it has another name,
   replace `app/build/outputs/...` in the YAML with `<yourmodule>/build/outputs/...`.
3. If your repo's default branch is `master`, change `main` to `master` in the YAML.
4. Your project must build locally first: `./gradlew assembleDebug`.

## Step 3 — Push to GitHub

```bash
git add .github/workflows/android-build.yml
git commit -m "Add Android CI/CD workflow"
git push origin main
```

## Step 4 — Download your debug APK

1. Open your repo on GitHub → **Actions** tab.
2. Click the latest **Android CI/CD** run.
3. Scroll to **Artifacts** → download **app-debug-apk** (a zip containing the APK).
4. Unzip, copy the APK to your phone, and install it
   (allow "Install unknown apps" when prompted).

---

## Step 5 (optional) — Signed release APK on GitHub Releases

A debug APK is fine for testing. For a proper release, sign it.

### 5a. Create a keystore (one time)

```bash
keytool -genkey -v -keystore release.keystore -alias mykey \
  -keyalg RSA -keysize 2048 -validity 10000
```

Remember the keystore password, alias, and key password.
**Back up this file — if you lose it you can't update your app.**
Never commit it to Git.

### 5b. Convert the keystore to base64

- Linux / macOS: `base64 -w 0 release.keystore > keystore.b64`
  (macOS: `base64 -i release.keystore -o keystore.b64`)
- Windows PowerShell:
  `[Convert]::ToBase64String([IO.File]::ReadAllBytes("release.keystore")) | Out-File keystore.b64`

### 5c. Add GitHub Secrets

Repo → **Settings → Secrets and variables → Actions → New repository secret**

| Secret name | Value |
|---|---|
| `KEYSTORE_BASE64` | contents of `keystore.b64` |
| `KEY_ALIAS` | your alias (e.g. `mykey`) |
| `KEYSTORE_PASSWORD` | keystore password |
| `KEY_PASSWORD` | key password |

### 5d. Publish a release

Repo → **Releases → Draft a new release** → choose/create a tag like `v1.0.0`
→ **Publish release**. After a few minutes the signed APK appears under the
release's **Assets**.

---

## Workflow permissions

If the release upload fails with "Resource not accessible by integration":
Repo → **Settings → Actions → General → Workflow permissions** →
select **Read and write permissions** → Save.

## Troubleshooting

| Problem | Fix |
|---|---|
| `gradlew: Permission denied` | Already handled by the `chmod +x` step |
| `No files were found` on upload | Check your module name / output path |
| `SDK location not found` | Not needed on GitHub runners; remove any hard-coded `local.properties` from Git |
| Java version errors | The workflow uses JDK 17. For older projects (AGP < 7) change `java-version` to `11` |
| Lint / test failures | Add `-x lint` to the Gradle command, or fix the reported issues |
| Build is slow the first time | Normal; Gradle caching speeds up later runs |

## Optional: run unit tests before building

Add this step before "Build debug APK":

```yaml
      - name: Run unit tests
        run: ./gradlew testDebugUnitTest
```
