I'll help you set up GitHub Actions to build and distribute your Flutter iOS app. Here's a comprehensive guide:

## Basic GitHub Actions Workflow

Create a file at `.github/workflows/ios-build.yml`:

```yaml
name: Build iOS IPA

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:  # Allows manual trigger

jobs:
  build-ios:
    runs-on: macos-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Set up Flutter
      uses: subosito/flutter-action@v2
      with:
        flutter-version: '3.19.0'  # Specify your version
        channel: 'stable'
    
    - name: Install dependencies
      run: flutter pub get
    
    - name: Build iOS (no code signing)
      run: flutter build ios --release --no-codesign
    
    - name: Create IPA manually
      run: |
        cd build/ios/iphoneos
        mkdir Payload
        cp -r Runner.app Payload/
        zip -r app.ipa Payload
    
    - name: Upload IPA as artifact
      uses: actions/upload-artifact@v3
      with:
        name: ios-ipa
        path: build/ios/iphoneos/app.ipa
```

## With Code Signing (Production Ready)

For a properly signed IPA that can be distributed:

```yaml
name: Build and Sign iOS IPA

on:
  push:
    branches: [ main ]
  workflow_dispatch:

jobs:
  build-ios:
    runs-on: macos-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Set up Flutter
      uses: subosito/flutter-action@v2
      with:
        flutter-version: '3.19.0'
        channel: 'stable'
    
    - name: Install dependencies
      run: flutter pub get
    
    - name: Import certificates
      env:
        CERTIFICATE_BASE64: ${{ secrets.IOS_CERTIFICATE_BASE64 }}
        P12_PASSWORD: ${{ secrets.IOS_CERTIFICATE_PASSWORD }}
      run: |
        # Create keychain
        security create-keychain -p actions build.keychain
        security default-keychain -s build.keychain
        security unlock-keychain -p actions build.keychain
        security set-keychain-settings -t 3600 -u build.keychain
        
        # Import certificate
        echo $CERTIFICATE_BASE64 | base64 --decode > certificate.p12
        security import certificate.p12 -k build.keychain -P $P12_PASSWORD -T /usr/bin/codesign
        security set-key-partition-list -S apple-tool:,apple: -s -k actions build.keychain
        
        rm certificate.p12
    
    - name: Import provisioning profile
      env:
        PROVISIONING_PROFILE_BASE64: ${{ secrets.IOS_PROVISIONING_PROFILE_BASE64 }}
      run: |
        mkdir -p ~/Library/MobileDevice/Provisioning\ Profiles
        echo $PROVISIONING_PROFILE_BASE64 | base64 --decode > ~/Library/MobileDevice/Provisioning\ Profiles/profile.mobileprovision
    
    - name: Build iOS
      run: flutter build ipa --release --export-options-plist=ios/ExportOptions.plist
    
    - name: Upload IPA to GitHub Artifacts
      uses: actions/upload-artifact@v3
      with:
        name: ios-release-ipa
        path: build/ios/ipa/*.ipa
        retention-days: 30
```

## Distribution Options

### 1. **GitHub Artifacts** (Already shown above)
Download from the Actions tab in your repository.

### 2. **Upload to TestFlight/App Store Connect**
```yaml
    - name: Upload to TestFlight
      env:
        APP_STORE_CONNECT_API_KEY: ${{ secrets.APP_STORE_CONNECT_API_KEY }}
      run: |
        xcrun altool --upload-app -f build/ios/ipa/*.ipa \
          -t ios --apiKey $APP_STORE_CONNECT_API_KEY
```

### 3. **Upload to Firebase App Distribution**
```yaml
    - name: Upload to Firebase App Distribution
      uses: wzieba/Firebase-Distribution-Github-Action@v1
      with:
        appId: ${{ secrets.FIREBASE_APP_ID }}
        token: ${{ secrets.FIREBASE_TOKEN }}
        groups: testers
        file: build/ios/ipa/*.ipa
```

### 4. **Upload to AWS S3**
```yaml
    - name: Upload to S3
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      run: |
        aws s3 cp build/ios/ipa/*.ipa s3://your-bucket/builds/$(date +%Y%m%d-%H%M%S).ipa
```

### 5. **Create GitHub Release**
```yaml
    - name: Create Release
      uses: softprops/action-gh-release@v1
      if: startsWith(github.ref, 'refs/tags/')
      with:
        files: build/ios/ipa/*.ipa
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Setting Up Secrets

You'll need to add these secrets in your GitHub repository settings (Settings → Secrets and variables → Actions):

1. **IOS_CERTIFICATE_BASE64**: Your .p12 certificate file encoded in base64
   ```bash
   base64 -i certificate.p12 | pbcopy
   ```

2. **IOS_CERTIFICATE_PASSWORD**: Password for the .p12 file

3. **IOS_PROVISIONING_PROFILE_BASE64**: Your provisioning profile encoded in base64
   ```bash
   base64 -i profile.mobileprovision | pbcopy
   ```

## ExportOptions.plist Example

Create `ios/ExportOptions.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>method</key>
    <string>app-store</string>
    <key>teamID</key>
    <string>YOUR_TEAM_ID</string>
    <key>uploadBitcode</key>
    <false/>
    <key>compileBitcode</key>
    <false/>
    <key>uploadSymbols</key>
    <true/>
</dict>
</plist>
```

Replace `method` with `ad-hoc`, `enterprise`, or `development` as needed.

The simplest approach for getting started is using GitHub Artifacts (first example), which lets you download the IPA directly from the Actions tab. For production apps, you'll want the signed version with proper distribution.