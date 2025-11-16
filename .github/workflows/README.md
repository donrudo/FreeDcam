# GitHub Actions Workflows

This directory contains automated workflows for building and releasing FreeDcam APKs.

## Available Workflows

### 1. Build APK (`build-apk.yml`)

**Purpose:** Automatically builds debug and release APKs on every push and pull request.

**Triggers:**
- Push to `master`, `main`, or `develop` branches
- Pull requests to `master`, `main`, or `develop` branches
- Manual trigger via "Actions" tab

**What it does:**
1. Sets up Java 11 and Android SDK/NDK
2. Builds both debug and release APKs
3. Uploads APKs as downloadable artifacts
4. Generates build summary

**Artifacts:**
- `FreeDcam-debug-{version}-{commit}.apk` (retained for 30 days)
- `FreeDcam-release-{version}-{commit}.apk` (retained for 90 days)

**How to download:**
1. Go to the "Actions" tab in GitHub
2. Click on the latest successful workflow run
3. Scroll down to "Artifacts" section
4. Download the desired APK

---

### 2. Build and Release APK (`release.yml`)

**Purpose:** Creates official releases with APK attachments when version tags are pushed.

**Triggers:**
- Push of version tags (e.g., `v4.3.82`)
- Manual trigger via "Actions" tab (with version input)

**What it does:**
1. Builds release APK
2. Generates changelog from commits since last tag
3. Creates GitHub Release with APK attached
4. Calculates checksums (MD5, SHA1, SHA256)

**How to create a release:**

#### Method 1: Tag-based (Recommended)
```bash
# Create and push a version tag
git tag v4.3.83
git push origin v4.3.83
```

The workflow will automatically:
- Build the release APK
- Generate changelog
- Create a GitHub Release
- Attach the APK to the release

#### Method 2: Manual Trigger
1. Go to "Actions" tab
2. Select "Build and Release APK" workflow
3. Click "Run workflow"
4. Enter version name (e.g., `4.3.83`)
5. Click "Run workflow"

This will build the APK and upload as an artifact (but won't create a GitHub Release).

---

### 3. CodeQL Analysis (`codeql-analysis.yml`)

**Purpose:** Security and code quality analysis.

**Triggers:**
- Push to `master` branch
- Pull requests to `master` branch
- Weekly schedule (Mondays at 20:18 UTC)

**Languages analyzed:**
- C/C++ (native code)
- Java (Android code)
- Python (build scripts)

---

## Build Requirements

### Gradle & Java
- **Gradle:** 7.2.1
- **Java:** 11 (Temurin distribution)
- **Gradle Build Tools:** 7.2.1

### Android SDK
- **Compile SDK:** 31 (Android 12)
- **Build Tools:** 30.0.3
- **Min SDK:** 14 (Android 4.0)
- **Target SDK:** 31 (Android 12)

### NDK
- **Version:** 21.4.7075529
- **ABIs:** armeabi-v7a, arm64-v8a, x86, x86_64
- **Native Libraries:** LibRaw, LibTIFF

### Dependencies
- Dagger Hilt 2.42
- AndroidX libraries
- RenderScript Intrinsics Replacement Toolkit

---

## Workflow Configuration

### Caching
Both workflows use Gradle dependency caching to speed up builds:
- Cache key based on gradle files
- Reduces build time significantly on subsequent runs

### Submodules
Workflows automatically checkout submodules (Camera1Parameters).

### Signing
- Uses existing keystore: `key/freedcamkey.jks`
- **Password:** freedcam (configured in `app/build.gradle`)
- Both debug and release builds are signed

> **Note:** For production releases, consider using GitHub Secrets to store signing credentials securely.

---

## Customization

### Changing Trigger Branches
Edit the workflow file:
```yaml
on:
  push:
    branches: [ master, main, develop, your-branch ]
```

### Adjusting Artifact Retention
```yaml
retention-days: 30  # Change to desired number of days
```

### Using Different NDK Version
```yaml
- name: Install NDK
  run: |
    echo "y" | sudo ${ANDROID_HOME}/cmdline-tools/latest/bin/sdkmanager --install "ndk;25.2.9519653"
```

### Secure Signing (Recommended for Production)

Add secrets in GitHub repository settings:
1. Go to Settings → Secrets and variables → Actions
2. Add these secrets:
   - `KEYSTORE_FILE` (base64 encoded keystore)
   - `KEYSTORE_PASSWORD`
   - `KEY_ALIAS`
   - `KEY_PASSWORD`

Then modify `build.gradle`:
```gradle
signingConfigs {
    release {
        if (System.getenv("CI")) {
            // Use GitHub secrets in CI
            storeFile file("release.jks")
            storePassword System.getenv("KEYSTORE_PASSWORD")
            keyAlias System.getenv("KEY_ALIAS")
            keyPassword System.getenv("KEY_PASSWORD")
        } else {
            // Use local keystore for development
            storeFile file('../key/freedcamkey.jks')
            storePassword 'freedcam'
            keyAlias 'freedcamkey'
            keyPassword 'freedcam'
        }
    }
}
```

---

## Troubleshooting

### Build Fails with "NDK not found"
The workflow automatically installs NDK 21.4.7075529. If using a different version locally, ensure the workflow NDK version matches your local setup.

### Build Fails with Gradle Errors
- Check Java version (must be 11 for Gradle 7.2.1)
- Clear Gradle cache: Delete `.gradle` cache in workflow
- Check for syntax errors in `build.gradle`

### Submodule Not Checked Out
Ensure workflow has:
```yaml
- uses: actions/checkout@v4
  with:
    submodules: recursive
```

### APK Not Uploaded
- Check that build succeeded
- Verify output path: `app/build/outputs/apk/release/`
- Check workflow logs for errors

---

## Best Practices

### For Development
1. Push to feature branches - APKs built automatically
2. Download debug APK from workflow artifacts for testing
3. Test on real devices before creating releases

### For Releases
1. Update version in `app/build.gradle`:
   ```gradle
   versionCode 283
   versionName '4.3.83'
   ```
2. Commit version bump
3. Create and push tag:
   ```bash
   git tag v4.3.83
   git push origin v4.3.83
   ```
4. Workflow creates release with APK automatically

### Security
- Don't commit signing keys to public repositories
- Use GitHub Secrets for sensitive credentials
- Enable branch protection for `master`/`main`
- Require PR reviews before merging

---

## Monitoring

### Build Status
- Check the "Actions" tab for workflow runs
- Green checkmark = successful build
- Red X = failed build (check logs)

### Email Notifications
GitHub sends email notifications for:
- Workflow failures
- First successful run after failures

Configure in: Settings → Notifications → Actions

---

## Additional Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Android CI/CD Best Practices](https://developer.android.com/studio/projects/continuous-integration)
- [Gradle Build Cache](https://docs.gradle.org/current/userguide/build_cache.html)

---

**Last Updated:** 2025-11-16
**Maintained by:** FreeDcam Development Team
