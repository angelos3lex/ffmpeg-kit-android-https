As described in https://medium.com/@nooruddinlakhani/resolved-ffmpegkit-retirement-issue-in-react-native-a-complete-guide-0f54b113b390

This provides a cached aar file that can be used for https-lts version of ffmpegkit.

### 1. You will need to apply `patches/ffmpeg-kit-react-native+6.0.2.patch` with patch-package:
```
diff --git a/node_modules/ffmpeg-kit-react-native/android/build.gradle b/node_modules/ffmpeg-kit-react-native/android/build.gradle
index 2909280..c535e2b 100644
--- a/node_modules/ffmpeg-kit-react-native/android/build.gradle
+++ b/node_modules/ffmpeg-kit-react-native/android/build.gradle
@@ -33,7 +33,8 @@ android {
   compileSdkVersion 33

   defaultConfig {
-    minSdkVersion safeExtGet('ffmpegKitPackage', 'https').contains("-lts") ? 16 : 24
+    // minSdkVersion safeExtGet('ffmpegKitPackage', 'https').contains("-lts") ? 16 : 24
+    minSdkVersion 16
     targetSdkVersion 33
     versionCode 602
     versionName "6.0.2"
@@ -125,5 +126,6 @@ repositories {

 dependencies {
   api 'com.facebook.react:react-native:+'
-  implementation 'com.arthenica:ffmpeg-kit-' + safePackageName(safeExtGet('ffmpegKitPackage', 'https')) + ':' + safePackageVersion(safeExtGet('ffmpegKitPackage', 'https'))
+  // implementation 'com.arthenica:ffmpeg-kit-' + safePackageName(safeExtGet('ffmpegKitPackage', 'https')) + ':' + safePackageVersion(safeExtGet('ffmpegKitPackage', 'https'))
+  implementation(name: 'ffmpeg-kit-https-lts', ext: 'aar')
 }
```

### 2. On your root `build.gradle` remove:
`ffmpegKitPackage = "https-lts"`
and add:
```
allprojects {
    repositories {
        ...
        flatDir {
            dirs "$rootDir/libs"
        }
```

### 3. On your app's `build.gradle`:
Add on top of the file:
`import java.net.URL`

Also add on end of `android { block`:
```
android {
    ...
    ...
    // For accessing the file locally after downloading
    repositories {
        flatDir {
            dirs "$rootDir/libs"
        }
    }
```

Then also add on `dependencies { block`:
```
dependencies {
    implementation(name: 'ffmpeg-kit-https-lts', ext: 'aar')
    implementation 'com.arthenica:smart-exception-java:0.2.1'
    ...
```

And finally also add (outside of any block):
```
// Add the following script to download the file from the cloud
afterEvaluate {
    def aarUrl = 'https://github.com/angelos3lex/ffmpeg-kit-android-https/releases/download/latest/ffmpeg-kit-https-lts.aar'
    def aarFile = file("${rootDir}/libs/ffmpeg-kit-https-lts.aar")

    tasks.register("downloadAar") {
        doLast {
             if (!aarFile.parentFile.exists()) {
                println "📁 Creating directory: ${aarFile.parentFile.absolutePath}"
                aarFile.parentFile.mkdirs()
            }
            if (!aarFile.exists()) {
                println "⏬ Downloading AAR from $aarUrl..."
                new URL(aarUrl).withInputStream { i ->
                    aarFile.withOutputStream { it << i }
                }
                println "✅ AAR downloaded to ${aarFile.absolutePath}"
            } else {
                println "ℹ️ AAR already exists at ${aarFile.absolutePath}"
            }
        }
    }

    // Make sure the AAR is downloaded before compilation begins
    preBuild.dependsOn("downloadAar")
}
```

### 4. Final step: Download the aar.

Either using a post-install hook in `package.json` like:
```
"scripts": {
    "postinstall": "cd android && ./gradlew :app:downloadAar"
  }
```

Or make sure you do:
`./gradlew :app:downloadAar` before any `assembleDebug` or `assembleRelease` command.

You may also want to add to your `.gitgnore`: `**/android/libs/` In order to not add on git the downloaded aar file.
