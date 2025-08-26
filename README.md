# Creating Android project files and zipping them for user download.
import os, textwrap, zipfile, pathlib, json

root = "/mnt/data/BastradGPSFake"
if os.path.exists(root):
    # remove existing to ensure fresh
    import shutil
    shutil.rmtree(root)
os.makedirs(root, exist_ok=True)

# File contents
files = {
    "settings.gradle": "rootProject.name = 'BastradGPSFake'\ninclude ':app'\n",
    "gradle.properties": "org.gradle.jvmargs=-Xmx1536m\nandroid.useAndroidX=true\n",
    "build.gradle": textwrap.dedent("""\
        // Top-level build file where you can add configuration options common to all sub-projects/modules.
        buildscript {
            repositories {
                google()
                mavenCentral()
            }
            dependencies {
                classpath 'com.android.tools.build:gradle:8.1.0'
                classpath 'org.jetbrains.kotlin:kotlin-gradle-plugin:1.9.10'
            }
        }
        
        allprojects {
            repositories {
                google()
                mavenCentral()
            }
        }
        
        task clean(type: Delete) {
            delete rootProject.buildDir
        }
    """),
    "app/build.gradle": textwrap.dedent("""\
        plugins {
            id 'com.android.application'
            id 'kotlin-android'
        }
        
        android {
            namespace 'com.bastrad.fakegps'
            compileSdk 34
        
            defaultConfig {
                applicationId 'com.bastrad.fakegps'
                minSdk 24
                targetSdk 34
                versionCode 1
                versionName '1.0'
            }
        
            buildTypes {
                release {
                    minifyEnabled false
                    proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
                }
            }
            compileOptions {
                sourceCompatibility JavaVersion.VERSION_17
                targetCompatibility JavaVersion.VERSION_17
            }
            kotlinOptions {
                jvmTarget = '17'
            }
        }
        
        dependencies {
            implementation 'androidx.core:core-ktx:1.12.0'
            implementation 'androidx.appcompat:appcompat:1.6.1'
            implementation 'com.google.android.material:material:1.11.0'
        }
    """),
    "app/proguard-rules.pro": "# Add your proguard rules here\n",
    "app/src/main/AndroidManifest.xml": textwrap.dedent("""\
        <?xml version=\"1.0\" encoding=\"utf-8\"?>
        <manifest xmlns:android=\"http://schemas.android.com/apk/res/android\"\n            package=\"com.bastrad.fakegps\">\n\n            <uses-permission android:name=\"android.permission.ACCESS_FINE_LOCATION\"/>\n            <uses-permission android:name=\"android.permission.ACCESS_COARSE_LOCATION\"/>\n            <uses-permission android:name=\"android.permission.INTERNET\"/>\n\n            <application\n                android:allowBackup=\"true\"\n                android:usesCleartextTraffic=\"true\"\n                android:icon=\"@mipmap/ic_launcher\"\n                android:label=\"BASTRAD GPS FAKE\"\n                android:theme=\"@style/Theme.AppCompat.Light.NoActionBar\">\n\n                <activity android:name=\"com.bastrad.fakegps.SplashActivity\"\n                    android:exported=\"true\"\n                    android:theme=\"@style/Theme.AppCompat.Light.NoActionBar\">\n                    <intent-filter>\n                        <action android:name=\"android.intent.action.MAIN\"/>\n                        <category android:name=\"android.intent.category.LAUNCHER\"/>\n                    </intent-filter>\n                </activity>\n\n                <activity android:name=\"com.bastrad.fakegps.MainActivity\"\n                    android:exported=\"false\" />\n\n            </application>\n        </manifest>\n    """),
    "app/src/main/java/com/bastrad/fakegps/MainActivity.kt": textwrap.dedent("""\
        package com.bastrad.fakegps
        
        import android.Manifest
        import android.content.pm.PackageManager
        import android.os.Bundle
        import android.webkit.GeolocationPermissions
        import android.webkit.WebChromeClient
        import android.webkit.WebSettings
        import android.webkit.WebView
        import android.webkit.WebViewClient
        import androidx.appcompat.app.AppCompatActivity
        import androidx.core.app.ActivityCompat
        import androidx.core.content.ContextCompat
        
        class MainActivity : AppCompatActivity() {
        
            private lateinit var webView: WebView
            private val LOCATION_PERMISSION_REQUEST = 1
        
            override fun onCreate(savedInstanceState: Bundle?) {
                super.onCreate(savedInstanceState)
                setContentView(R.layout.activity_main)
        
                // Request location permission if not granted
                if (ContextCompat.checkSelfPermission(this, Manifest.permission.ACCESS_FINE_LOCATION)
                    != PackageManager.PERMISSION_GRANTED
                ) {
                    ActivityCompat.requestPermissions(
                        this,
                        arrayOf(Manifest.permission.ACCESS_FINE_LOCATION, Manifest.permission.ACCESS_COARSE_LOCATION),
                        LOCATION_PERMISSION_REQUEST
                    )
                }
        
                webView = findViewById(R.id.webView)
                val settings: WebSettings = webView.settings
                settings.javaScriptEnabled = true
                settings.domStorageEnabled = true
                settings.allowFileAccess = true
                settings.allowContentAccess = true
                settings.setGeolocationEnabled(true)
        
                webView.webViewClient = WebViewClient()
                webView.webChromeClient = object : WebChromeClient() {
                    override fun onGeolocationPermissionsShowPrompt(origin: String?, callback: GeolocationPermissions.Callback?) {
                        callback?.invoke(origin, true, false)
                    }
                }
        
                // Load local web app (copy your dist/ into assets/dist/)
                webView.loadUrl(\"file:///android_asset/dist/index.html\")
            }
        
            override fun onBackPressed() {
                if (webView.canGoBack()) webView.goBack() else super.onBackPressed()
            }
        }
    """),
    "app/src/main/java/com/bastrad/fakegps/SplashActivity.kt": textwrap.dedent("""\
        package com.bastrad.fakegps
        
        import android.content.Intent
        import android.os.Bundle
        import android.os.Handler
        import android.os.Looper
        import androidx.appcompat.app.AppCompatActivity
        
        class SplashActivity : AppCompatActivity() {
            override fun onCreate(savedInstanceState: Bundle?) {
                super.onCreate(savedInstanceState)
                setContentView(R.layout.activity_splash)
        
                Handler(Looper.getMainLooper()).postDelayed({
                    startActivity(Intent(this, MainActivity::class.java))
                    finish()
                }, 1500)
            }
        }
    """),
    "app/src/main/res/layout/activity_main.xml": textwrap.dedent("""\
        <?xml version="1.0" encoding="utf-8"?>
        <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
            android:orientation="vertical"
            android:layout_width="match_parent"
            android:layout_height="match_parent">
        
            <androidx.appcompat.widget.Toolbar
                android:id="@+id/toolbar"
                android:layout_width="match_parent"
                android:layout_height="?attr/actionBarSize"
                android:background="#1abc9c"
                android:title="BASTRAD GPS FAKE"
                android:titleTextColor="@android:color/white"/>
        
            <WebView
                android:id="@+id/webView"
                android:layout_width="match_parent"
                android:layout_height="match_parent" />
        </LinearLayout>
    """),
    "app/src/main/res/layout/activity_splash.xml": textwrap.dedent("""\
        <?xml version=\"1.0\" encoding=\"utf-8\"?>
        <RelativeLayout xmlns:android=\"http://schemas.android.com/apk/res/android\"\n            android:layout_width=\"match_parent\"\n            android:layout_height=\"match_parent\"\n            android:gravity=\"center\"\n            android:background=\"#ffffff\">\n\n            <ImageView\n                android:id=\"@+id/splashIcon\"\n                android:layout_width=\"100dp\"\n                android:layout_height=\"100dp\"\n                android:layout_centerInParent=\"true\"\n                android:src=\"@mipmap/ic_launcher\"/>\n\n            <TextView\n                android:layout_width=\"wrap_content\"\n                android:layout_height=\"wrap_content\"\n                android:text=\"BASTRAD GPS FAKE\"\n                android:textSize=\"20sp\"\n                android:textStyle=\"bold\"\n                android:layout_below=\"@id/splashIcon\"\n                android:layout_centerHorizontal=\"true\"\n                android:layout_marginTop=\"20dp\"/>\n        </RelativeLayout>
    """),
    "app/src/main/assets/dist/index.html": textwrap.dedent("""\
        <!doctype html>\n        <html>\n        <head>\n          <meta charset=\"utf-8\">\n          <meta name=\"viewport\" content=\"width=device-width, initial-scale=1\">\n          <title>BASTRAD GPS FAKE - Dummy</title>\n        </head>\n        <body>\n          <h1>BASTRAD GPS FAKE</h1>\n          <p>This is a placeholder web build. Replace this folder with your real <code>dist/</code> from the web project.</p>\n        </body>\n        </html>\n    """),
    "app/src/main/assets/dist/dummy.js": "console.log('dummy');\n",
    "app/src/main/res/values/strings.xml": textwrap.dedent("""\
        <?xml version=\"1.0\" encoding=\"utf-8\"?>\n        <resources>\n            <string name=\"app_name\">BASTRAD GPS FAKE</string>\n        </resources>\n    """),
    "app/src/main/res/values/styles.xml": textwrap.dedent("""\
        <?xml version=\"1.0\" encoding=\"utf-8\"?>\n        <resources>\n            <style name=\"Theme.App\" parent=\"Theme.AppCompat.Light.NoActionBar\">\n                <!-- Customize your theme here. -->\n            </style>\n        </resources>\n    """)
}

# create files
for path, content in files.items():
    full = os.path.join(root, path)
    os.makedirs(os.path.dirname(full), exist_ok=True)
    with open(full, "w", encoding="utf-8") as f:
        f.write(content)

# create a simple README
readme = textwrap.dedent(f"""\
    BASTRAD GPS FAKE - Android WebView Template\n\n
    Package name: com.bastrad.fakegps\n\n
    Instructions:\n    1. Build your web app and copy the entire contents of its dist/ into app/src/main/assets/dist/\n    2. Open this project in Android Studio.\n    3. Build > Build Bundle(s) / APK(s) > Build APK(s).\n    4. Install the generated APK on your device.\n\n    Note: This template includes a placeholder dist/index.html -- replace with your real build.\n""")
with open(os.path.join(root, "README.txt"), "w", encoding="utf-8") as f:
    f.write(readme)

# zip it
zip_path = "/mnt/data/BastradGPSFake.zip"
with zipfile.ZipFile(zip_path, "w", zipfile.ZIP_DEFLATED) as z:
    for folder, subs, files_in in os.walk(root):
        for file in files_in:
            file_path = os.path.join(folder, file)
            arcname = os.path.relpath(file_path, root)
            z.write(file_path, arcname)

# show result
{"zip_path": zip_path, "size_bytes": os.path.getsize(zip_path), "files_created": sum(len(files) for files in os.walk(root))}

