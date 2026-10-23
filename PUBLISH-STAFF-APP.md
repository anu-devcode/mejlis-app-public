# Make the Mejlis staff app available: complete checklist

Use `anu-devcode/mejlis-web` for private source and builds. Use `anu-devcode/mejlis-app-public` only for public APK releases. Android Studio is not required. The new public repository in your screenshot is empty: it needs an initial README commit before a release can be created.

## 1. Prepare the two repositories

1. In mejlis-app-public, click Add a README (or create README.md), write a short app description, and commit to main. Do not copy private source or signing files there.
2. Commit and push the implementation changes in your existing private mejlis-web repository to main. This includes the Android APK workflow, capture policy migration/plugin, icon generator, frontend and backend changes.
3. Leave the private source repository private. A downloadable app does not require exposing its source.

## 2. Understand and check your release signing key

A signing key is a private cryptographic key that Android uses to recognize your app and accept later updates from the same publisher. It is stored in a password-protected keystore (.jks or .keystore). It is NOT a Telegram bot token, a webhook secret, a GitHub token, or your account password.

If you have an existing release keystore, preserve and use it. Check your own saved signing folder/backups and GitHub repository Settings → Secrets and variables → Actions for the four ANDROID_KEYSTORE/KEY secrets below. GitHub shows secret names, not their original values. Do not overwrite an existing key until you know whether staff already use a release signed with it.

The previously produced debug APK uses a test signing key. A hosted runner's debug key is not a durable release key. You cannot recover a private release signing key from an APK. Its public certificate/fingerprint is not the private key.

No java/keytool command was available on PATH in this workspace during implementation. Install a JDK (not just a JRE), for example Eclipse Temurin JDK 21 from https://adoptium.net/ . Android Studio is unnecessary. Enable the installer's PATH option, then open a new PowerShell window and check:

```powershell
java -version
keytool -help
```

Create a private signing folder outside your Git repository (for example C:\MejlisSigning). If you do NOT already have a release key, run inside that folder:

```powershell
keytool -genkeypair -v -keystore mejlis-release.jks -storetype PKCS12 -alias mejlis -keyalg RSA -keysize 3072 -validity 10000
```

Enter a strong, unique password at the prompt and complete the certificate details. Store the password in your password manager. Back up the keystore privately/offline. Do not generate another key for each update.

Check an existing or newly created key (the password is entered interactively):

```powershell
keytool -list -v -keystore .\mejlis-release.jks -alias mejlis
```

Confirm it is a PrivateKeyEntry and record its SHA-256 certificate fingerprint for your release records. Never share the private key or passwords in chat. If an existing key's alias is unknown, omit `-alias mejlis` to list its entries.

## 3. Add signing secrets to the PRIVATE source repository

From the signing folder, copy the Base64-encoded keystore to your clipboard:

```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes((Resolve-Path .\mejlis-release.jks))) | Set-Clipboard
```

In GitHub → anu-devcode/mejlis-web → Settings → Secrets and variables → Actions → New repository secret, add:

| Name | Value |
| --- | --- |
| ANDROID_KEYSTORE_BASE64 | Paste the clipboard's Base64 text |
| ANDROID_KEYSTORE_PASSWORD | Your keystore password |
| ANDROID_KEY_ALIAS | mejlis (or your existing key's actual alias) |
| ANDROID_KEY_PASSWORD | The private-key password, normally the same password for the generated PKCS12 key |

Clear your clipboard afterward. Do not upload the keystore to a Release or commit it to either repository.

## 4. Choose manual upload or automatic public release draft

Manual upload needs no GitHub publishing token: download the build artifact ZIP, extract it yourself and attach the three individual files in step 7.

For automatic draft upload, create a fine-grained GitHub personal access token from your GitHub account Settings → Developer settings → Personal access tokens → Fine-grained tokens. Set its resource owner to anu-devcode, restrict repository access to ONLY mejlis-app-public, give Repository permissions → Contents: Read and write, and set an expiry. Save the token as ANDROID_PUBLISH_TOKEN in the PRIVATE mejlis-web repository's Actions secrets. It does not need write access to your source repository. Renew it before expiry. Do not paste it into a VITE variable or a public file.

## 5. Configure Coolify and deploy the updated services

In the production backend environment:

```env
ANDROID_RELEASE_REPO=anu-devcode/mejlis-app-public
ANDROID_RELEASE_TOKEN=
```

An empty ANDROID_RELEASE_TOKEN works for public releases subject to GitHub's unauthenticated API limits. If those limits are reached, use a separate token with Contents READ access to the downloads repository only. The write-enabled ANDROID_PUBLISH_TOKEN belongs in GitHub Actions, not in the frontend.

Back up your database before deploying. The added migration `20261023090000_capture_override` adds users.screenshot_allowed with default false. No existing records are deleted and no accounts receive permission by default. The normal backend startup runs `prisma migrate deploy`; verify successful migration in Coolify deployment logs. Rebuild/deploy BOTH backend and frontend from the updated main branch. If your deployment bypasses normal startup, run migrations in the backend environment yourself before starting the new code.

The download will show unavailable until a stable public release with the required files exists. No production deployment, migration, secret configuration or release publication was performed during local implementation.

## 6. Build the signed APK on GitHub

In the PRIVATE mejlis-web repository: Actions → Android APK → Run workflow:

- Branch: main with the current changes.
- build_type: release (not debug).
- version_code: an integer higher than EVERY build you intend this release to update (for example 3 after build 2).
- version_name: the displayed version, for example 1.0.1.
- publish_draft: check it only if step 4's publishing token is configured. Leave unchecked for manual upload.

Run the workflow. It uses the official logo, builds the APK, signs and verifies it, and generates a matching staff-app.json. An artifact ZIP is retained for owner convenience; staff download the raw APK, not that ZIP.

If signing secrets are missing, the release step fails with an explicit message. If SDK setup fails on an old commit requesting the obsolete tools package, start a NEW run on the updated branch, not Re-run on the old failed commit.

## 7. Publish the public release

If publish_draft was checked, open mejlis-app-public → Releases and review the generated draft. Otherwise download the successful run's artifact ZIP, extract it, then create a release manually in mejlis-app-public with a new tag such as android-v1.0.1-3. Target the public repository's main branch.

Attach or verify all three raw assets:

1. mejlis-staff-id.apk
2. staff-app.json
3. SHA256SUMS.txt

Do not attach only the ZIP or rename it to .apk. Check that version_code in staff-app.json matches your workflow input. Test the signed APK on a test phone before public rollout; use the source workflow artifact to test before publishing the draft. When ready, publish it as a stable release and set it as latest. Drafts and pre-releases are not served by the production latest-release endpoint. Publishing makes this app binary publicly downloadable; it contains no signing key or backend secret.

## 8. Test the profile download and install

Open https://addisketemamejlis.org.et/api/v1/staff-app/android : expect JSON with the version and download URL, not a 503 error.

Sign in as authorized staff/admin → Profile or My Profile & ID → Download Android APK. Expect mejlis-staff-id.apk directly. The feedback says download requested; check the browser's download list to verify completion. Open the APK on Android and approve installation from the browser/files app if prompted. Staff do not need GitHub accounts.

If the phone has the older debug APK, Android may reject the signed release because its signing key differs. Uninstall the test APK first, then install the release. This clears local app/session data but does not delete your council account or server records. Future releases must keep the same application ID and release signing key and increase version_code.

## 9. Manage a screenshot exception

Install the new APK containing CapturePolicy support. Older APKs always keep FLAG_SECURE enabled and cannot gain this feature from a web-only deployment.

Sign in on the website with a true superadmin account (database isSuperuser=true) and assigned settings permission. Open Settings → Access/security tab, or the Security page. In Android screenshot exceptions, select the specific staff account, choose Allow or block, enter an audit reason and save. An ADMIN role name alone is not sufficient; the backend checks the actual database flag.

For the selected user, the updated app refreshes policy while online every 30 seconds. Exceptions have a 60-second native lease. Turning off capture protection permits BOTH screenshots and screen recording. Backgrounding, logout, failed policy refresh or lease expiry restores protection. When the same user returns and is still authorized, the exception can be renewed. Revoking the exception takes effect within the next refresh/lease window. Other users remain protected. This setting does not control Telegram, ordinary browsers or iOS capture behavior.

Test one allowed and one blocked account, then revoke the allowed user's exception and repeat. Android compilation and physical-device capture behavior must be verified on the new signed build; a successful web build cannot establish that behavior.

## 10. Release later updates

Repeat steps 6 and 7 using the SAME signing key and a higher version_code. No new frontend download URL is needed: the backend discovers the public repository's latest stable release. The app checks on launch, resume and every five minutes, and blocks outdated clients that contain the update checker. Backend release metadata is cached for up to five minutes without a token, or one minute with a token. The first APK with the checker must be installed manually. Android still requires approval to install a sideloaded update; silent background installation is not implemented.

If the profile shows Download unavailable: verify the public repository name, a published/latest stable release, the exact three asset filenames and a matching manifest. A GitHub ZIP-only release cannot work. If APK installation fails, check the signing key, version_code and that the file was fully downloaded. Share the failed workflow step's error text for any remaining build error; never include secret values.
