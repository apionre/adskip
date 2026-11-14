name: build
on: [push, workflow_dispatch]
jobs:
  apk:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: 17 }
      - uses: gradle/actions/setup-gradle@v3
        with: { gradle-version: 8.7 }
      - name: 生成工程文件
        run: |
          mkdir -p app/src/main/java/com/example/adskip app/src/main/res/xml
          cat > settings.gradle.kts <<'EOF'
          pluginManagement {
              repositories { google(); mavenCentral(); gradlePluginPortal() }
              plugins {
                  id("com.android.application") version "8.5.2"
                  id("org.jetbrains.kotlin.android") version "1.9.24"
              }
          }
          dependencyResolutionManagement {
              repositories { google(); mavenCentral() }
          }
          rootProject.name = "AdSkip"
          include(":app")
          EOF
          cat > app/build.gradle.kts <<'EOF'
          plugins {
              id("com.android.application")
              id("org.jetbrains.kotlin.android")
          }
          android {
              namespace = "com.example.adskip"
              compileSdk = 34
              defaultConfig {
                  applicationId = "com.example.adskip"
                  minSdk = 24
                  targetSdk = 34
                  versionCode = 1
                  versionName = "1.0"
              }
              compileOptions {
                  sourceCompatibility = JavaVersion.VERSION_17
                  targetCompatibility = JavaVersion.VERSION_17
              }
              kotlinOptions { jvmTarget = "17" }
          }
          EOF
          cat > app/src/main/AndroidManifest.xml <<'EOF'
          <?xml version="1.0" encoding="utf-8"?>
          <manifest xmlns:android="http://schemas.android.com/apk/res/android">
              <application android:label="广告跳过">
                  <activity android:name=".MainActivity" android:exported="true">
                      <intent-filter>
                          <action android:name="android.intent.action.MAIN" />
                          <category android:name="android.intent.category.LAUNCHER" />
                      </intent-filter>
                  </activity>
                  <service
                      android:name=".SkipService"
                      android:exported="true"
                      android:label="广告跳过"
                      android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE">
                      <intent-filter>
                          <action android:name="android.accessibilityservice.AccessibilityService" />
                      </intent-filter>
                      <meta-data
                          android:name="android.accessibilityservice"
                          android:resource="@xml/accessibility_config" />
                  </service>
              </application>
          </manifest>
          EOF
          cat > app/src/main/res/xml/accessibility_config.xml <<'EOF'
          <accessibility-service xmlns:android="http://schemas.android.com/apk/res/android"
              android:accessibilityEventTypes="typeWindowStateChanged|typeWindowContentChanged"
              android:accessibilityFeedbackType="feedbackGeneric"
              android:canRetrieveWindowContent="true"
              android:notificationTimeout="100" />
          EOF
          cat > app/src/main/java/com/example/adskip/SkipService.kt <<'EOF'
          package com.example.adskip

          import android.accessibilityservice.AccessibilityService
          import android.os.SystemClock
          import android.view.accessibility.AccessibilityEvent
          import android.view.accessibility.AccessibilityNodeInfo

          class SkipService : AccessibilityService() {
              private var last = 0L

              override fun onAccessibilityEvent(e: AccessibilityEvent?) {
                  val type = e?.eventType ?: return
                  if (type != AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED &&
                      type != AccessibilityEvent.TYPE_WINDOW_CONTENT_CHANGED) return
                  val now = SystemClock.uptimeMillis()
                  if (now - last < 300) return
                  val root = rootInActiveWindow ?: return
                  if (root.packageName == packageName) return
                  for (kw in KEYWORDS) {
                      for (n in root.findAccessibilityNodeInfosByText(kw)) {
                          val txt = n.text?.toString() ?: continue
                          if (txt.length > 8) continue
                          if (clickUp(n)) { last = now; return }
                      }
                  }
              }

              private fun clickUp(node: AccessibilityNodeInfo): Boolean {
                  var c: AccessibilityNodeInfo? = node
                  var depth = 0
                  while (c != null && depth < 4) {
                      if (c.isClickable) return c.performAction(AccessibilityNodeInfo.ACTION_CLICK)
                      c = c.parent
                      depth++
                  }
                  return false
              }

              override fun onInterrupt() {}

              companion object {
                  val KEYWORDS = listOf("跳过", "跳過", "Skip")
              }
          }
          EOF
          cat > app/src/main/java/com/example/adskip/MainActivity.kt <<'EOF'
          package com.example.adskip

          import android.app.Activity
          import android.content.Intent
          import android.os.Bundle
          import android.provider.Settings
          import android.view.Gravity
          import android.widget.Button
          import android.widget.LinearLayout
          import android.widget.TextView

          class MainActivity : Activity() {
              override fun onCreate(savedInstanceState: Bundle?) {
                  super.onCreate(savedInstanceState)
                  val box = LinearLayout(this).apply {
                      orientation = LinearLayout.VERTICAL
                      gravity = Gravity.CENTER
                      setPadding(64, 64, 64, 64)
                  }
                  box.addView(TextView(this).apply {
                      text = "点击下方按钮，在无障碍里找到“广告跳过”并开启。开启后，打开其他App时会自动点击“跳过”按钮。"
                      textSize = 16f
                  })
                  box.addView(Button(this).apply {
                      text = "去开启无障碍服务"
                      setOnClickListener { startActivity(Intent(Settings.ACTION_ACCESSIBILITY_SETTINGS)) }
                  })
                  setContentView(box)
              }
          }
          EOF
      - run: gradle assembleDebug
      - uses: actions/upload-artifact@v4
        with:
          name: adskip-apk
          path: app/build/outputs/apk/debug/app-debug.apk