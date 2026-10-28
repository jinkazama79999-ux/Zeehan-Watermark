# AutoMark QuickShare — Struktur Proyek & Source Code Lengkap (Kotlin, API 26–35)

Dokumen teknis ini menyajikan struktur proyek lengkap dan kode sumber produksi (*production-ready source code*) untuk aplikasi Android mandiri **AutoMark QuickShare**. Aplikasi bekerja **100% secara luring (offline)** sebagai *background watcher* tanpa memodifikasi aplikasi kamera bawaan perangkat—kompatibel mulai dari **Android 8.0 Oreo (Min SDK 26, seperti Oppo Reno 7)** hingga **Android 14 / Android 15 (Target/Compile SDK 34/35, seperti Oppo Reno 13, Reno 14 5G, Samsung Galaxy OneUI, dan Xiaomi MIUI/HyperOS)**.

## 1. Struktur Pohon Proyek (Clean Architecture)

```text
AutoMarkQuickShare/
├── app/
│   ├── build.gradle.kts
│   └── src/main/
│       ├── AndroidManifest.xml
│       ├── assets/
│       │   ├── logo_default.png
│       │   └── offline_cities_bbox.json
│       └── java/com/automark/quickshare/
│           ├── domain/
│           │   └── model/
│           │       ├── MediaCaptureEvent.kt
│           │       └── WatermarkTemplate.kt
│           ├── data/
│           │   ├── preferences/
│           │   │   └── WatermarkTemplateRepository.kt
│           │   ├── location/
│           │   │   └── OfflineGeolocationManager.kt
│           │   └── media/
│           │       ├── UniversalMediaContentObserver.kt
│           │       └── WatermarkRenderer.kt
│           ├── service/
│           │   ├── AutoMarkWatcherService.kt
│           │   ├── QuickShareNotificationHelper.kt
│           │   └── OemBackgroundKillerHelper.kt
│           └── ui/
│               └── MainActivity.kt
```

| Layer & Modul | Komponen Utama | Spesifikasi Teknis & Kompatibilitas API 26–35 |
| :--- | :--- | :--- |
| **Media Watcher** | `UniversalMediaContentObserver.kt` | Memantau `MediaStore.Images` & `MediaStore.Video` via [ContentObserver](https://developer.android.com/reference/android/database/ContentObserver), *debounce delay* 700 ms (rentang 500–1000 ms), validasi ukuran byte konstan untuk mencegah *IO write lock*, serta filter *anti-loop* (`DCIM/AutoMark` dan prefix `AutoMark_`). |
| **Watermark Engine (Foto)** | `WatermarkRenderer.kt` | Dekode & kanvas `Bitmap.Config.ARGB_8888` berpresisi proporsional (12 MP, 50 MP, 64 MP, hingga 108 MP pada rasio 1:1, 4:3, 16:9, 9:16), penyalinan penuh tag [ExifInterface](https://developer.android.com/reference/androidx/exifinterface/media/ExifInterface), serta penyimpanan kompatibel ganda (`RELATIVE_PATH` + `IS_PENDING` di API 29+ dan `File` + [MediaScannerConnection](https://developer.android.com/reference/android/media/MediaScannerConnection) di API 26–28). |
| **Watermark Engine (Video)** | `WatermarkRenderer.kt` | Pipeline OpenGL/GPU hardware-accelerated via [AndroidX Media3 Transformer](https://developer.android.com/reference/androidx/media3/transformer/Transformer) dan [OverlayEffect](https://developer.android.com/reference/androidx/media3/effect/package-summary) (`BitmapOverlay` + `StaticOverlaySettings`) yang menjaga resolusi dan *framerate* asli (30/60 FPS). |
| **Offline Geolocation** | `OfflineGeolocationManager.kt` | Ekstraksi GPS dari EXIF foto/video dengan fallback pasif ke [FusedLocationProviderClient.getLastLocation()](https://developer.android.com/develop/sensors-and-location/location/retrieve-current) & `LocationManager`, dipetakan ke nama kota/kabupaten melalui [SQLiteDatabase](https://developer.android.com/reference/android/database/sqlite/SQLiteDatabase) *bounding-box* + rumus *Haversine* 100% offline. |
| **Template & Custom Builder** | `WatermarkTemplate.kt` & `WatermarkTemplateRepository.kt` | `data class WatermarkTemplate` dengan serialisasi JSON (`org.json.JSONObject`) ke `DataStore` / `SharedPreferences`, mencakup 4 preset bawaan (*Minimalist Corner*, *Modern Pill / Badge*, *Flagship / Studio Footer*, *Cinematic Letterbox*) dan *Custom Builder* (`TOP_LEFT`, `TOP_RIGHT`, `BOTTOM_LEFT`, `BOTTOM_RIGHT`, `BOTTOM_BANNER`). |
| **Service, OEM Helper & Quick Share** | `AutoMarkWatcherService.kt`, `QuickShareNotificationHelper.kt`, `OemBackgroundKillerHelper.kt` | [Foreground Service](https://developer.android.com/develop/background-work/services/fgs/service-types) persisten, panduan Intent *Auto-Launch / Battery Saver* untuk ColorOS (Oppo/Realme), MIUI/HyperOS (Xiaomi), dan OneUI (Samsung), serta *Heads-Up Notification* ([NotificationCompat](https://developer.android.com/reference/androidx/core/app/NotificationCompat.Builder)) untuk kirim instan ke IG Stories, WhatsApp, dan TikTok. |

## 2. Konfigurasi Gradle (`app/build.gradle.kts`)

Mendukung `minSdk = 26` (Android 8.0 Oreo) hingga `targetSdk = 35` / `compileSdk = 35` (Android 14 / 15).

```kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("kotlin-parcelize")
}

android {
    namespace = "com.automark.quickshare"
    compileSdk = 35

    defaultConfig {
        applicationId = "com.automark.quickshare"
        minSdk = 26 // Android 8.0 Oreo (Oppo Reno 7, seri lama, hingga Android 15 terbaru)
        targetSdk = 35 // Android 15 (Oppo Reno 13, Reno 14 5G, OneUI 7, HyperOS)
        versionCode = 1
        versionName = "1.0.0"

        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            isMinifyEnabled = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }

    kotlinOptions {
        jvmTarget = "17"
    }

    buildFeatures {
        compose = true
    }

    composeOptions {
        kotlinCompilerExtensionVersion = "1.5.15"
    }
}

dependencies {
    val media3Version = "1.5.0"
    val coroutinesVersion = "1.9.0"

    // Core AndroidX, Lifecycle Service & DataStore Preferences
    implementation("androidx.core:core-ktx:1.15.0")
    implementation("androidx.appcompat:appcompat:1.7.0")
    implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.8.7")
    implementation("androidx.lifecycle:lifecycle-service:2.8.7")
    implementation("androidx.datastore:datastore-preferences:1.1.1")

    // Jetpack Compose (Material 3) untuk UI Custom Template Builder
    implementation(platform("androidx.compose:compose-bom:2024.10.00"))
    implementation("androidx.activity:activity-compose:1.9.3")
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.material3:material3")
    implementation("androidx.compose.ui:ui-tooling-preview")

    // Coroutines untuk Asynchronous IO, Debouncing, dan Queue Processing
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:$coroutinesVersion")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:$coroutinesVersion")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-play-services:$coroutinesVersion")

    // AndroidX ExifInterface untuk duplikasi metadata EXIF & ekstraksi GPS
    implementation("androidx.exifinterface:exifinterface:1.3.7")

    // AndroidX Media3 Transformer & Effect untuk Hardware-Accelerated Video Watermarking (30/60 FPS)
    implementation("androidx.media3:media3-transformer:$media3Version")
    implementation("androidx.media3:media3-effect:$media3Version")
    implementation("androidx.media3:media3-common:$media3Version")
    implementation("androidx.media3:media3-muxer:$media3Version")

    // Google Play Services Location untuk fallback Last Known Location secara pasif/offline
    implementation("com.google.android.gms:play-services-location:21.3.0")

    // SQLite KTX untuk Offline Reverse Geocoding Database
    implementation("androidx.sqlite:sqlite-ktx:2.4.0")
    implementation("androidx.sqlite:sqlite-framework:2.4.0")
}
```

## 3. Konfigurasi `AndroidManifest.xml`

Menangani izin penyimpanan lintas generasi Android (API 26–28 `WRITE_EXTERNAL_STORAGE`, API 29–32 `READ_EXTERNAL_STORAGE`, dan API 33–35 `READ_MEDIA_IMAGES` / `READ_MEDIA_VIDEO`), izin lokasi EXIF & *Last Known Location*, izin [Foreground Service](https://developer.android.com/develop/background-work/services/fgs/service-types), serta `<queries>` untuk *Quick Share* ke Instagram Stories, WhatsApp, dan TikTok.

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    package="com.automark.quickshare">

    <!-- 1. Izin Media Android 13+ (API 33 - 35) -->
    <uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
    <uses-permission android:name="android.permission.READ_MEDIA_VIDEO" />
    <uses-permission android:name="android.permission.READ_MEDIA_VISUAL_USER_SELECTED" />

    <!-- 2. Izin Media Legacy Android 8.0 - 12 (API 26 - 32) -->
    <uses-permission
        android:name="android.permission.READ_EXTERNAL_STORAGE"
        android:maxSdkVersion="32" />
    <uses-permission
        android:name="android.permission.WRITE_EXTERNAL_STORAGE"
        android:maxSdkVersion="29"
        tools:ignore="ScopedStorage" />

    <!-- 3. Izin Lokasi Offline (Membaca EXIF GPS & Fallback FusedLocationProviderClient) -->
    <uses-permission android:name="android.permission.ACCESS_MEDIA_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />

    <!-- 4. Izin Foreground Service & Heads-Up Notification -->
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PROCESSING" />

    <!-- 5. Izin Anti-Background Killer (ColorOS / MIUI / OneUI) & Auto-Boot -->
    <uses-permission android:name="android.permission.REQUEST_IGNORE_BATTERY_OPTIMIZATIONS" />
    <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
    <uses-permission android:name="oppo.permission.OPPO_COMPONENT_SAFE" />

    <!-- 6. Package Visibility Queries (Android 11+) untuk Quick Share -->
    <queries>
        <!-- Target Aplikasi Sosial Media -->
        <package android:name="com.instagram.android" />
        <package android:name="com.whatsapp" />
        <package android:name="com.whatsapp.w4b" />
        <package android:name="com.ss.android.ugc.trill" />
        <package android:name="com.zhiliaoapp.musically" />

        <!-- Target OEM Security / Battery Center (ColorOS, MIUI, OneUI, Vivo) -->
        <package android:name="com.coloros.safecenter" />
        <package android:name="com.oplus.safecenter" />
        <package android:name="com.miui.securitycenter" />
        <package android:name="com.samsung.android.lool" />
        <package android:name="com.iqoo.secure" />

        <intent>
            <action android:name="android.intent.action.SEND" />
            <data android:mimeType="image/*" />
        </intent>
        <intent>
            <action android:name="android.intent.action.SEND" />
            <data android:mimeType="video/*" />
        </intent>
    </queries>

    <application
        android:allowBackup="true"
        android:label="AutoMark QuickShare"
        android:requestLegacyExternalStorage="true"
        android:largeHeap="true"
        android:supportsRtl="true"
        android:theme="@style/Theme.AppCompat.DayNight.NoActionBar">

        <!-- Activity Utama: Pengaturan Watcher, OEM Battery Guide, dan Custom Template Builder -->
        <activity
            android:name=".ui.MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <!-- Foreground Service Persisten -->
        <service
            android:name=".service.AutoMarkWatcherService"
            android:enabled="true"
            android:exported="false"
            android:foregroundServiceType="specialUse|mediaProcessing">
            <property
                android:name="android.app.PROPERTY_SPECIAL_USE_FGS_SUBTYPE"
                android:value="Persistent offline MediaStore ContentObserver for automatic camera watermarking and Quick Share notification." />
        </service>

        <!-- BroadcastReceiver untuk Tombol Pintasan Quick Share di Notifikasi -->
        <receiver
            android:name=".service.QuickShareActionReceiver"
            android:enabled="true"
            android:exported="false" />

        <!-- BroadcastReceiver untuk Menyalakan Ulang Service saat Perangkat Reboot -->
        <receiver
            android:name=".service.BootCompletedReceiver"
            android:enabled="true"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.BOOT_COMPLETED" />
                <action android:name="android.intent.action.QUICKBOOT_POWERON" />
                <action android:name="com.htc.intent.action.QUICKBOOT_POWERON" />
            </intent-filter>
        </receiver>

        <!-- FileProvider untuk Kompatibilitas Berbagi URI di API 26-28 -->
        <provider
            android:name="androidx.core.content.FileProvider"
            android:authorities="${applicationId}.fileprovider"
            android:exported="false"
            android:grantUriPermissions="true">
            <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                android:resource="@xml/file_paths" />
        </provider>

    </application>
</manifest>
```

> **Catatan `res/xml/file_paths.xml`** (untuk kompatibilitas `FileProvider` pada Android 8.0–9.0):
```xml
<?xml version="1.0" encoding="utf-8"?>
<paths>
    <external-path name="external_storage" path="." />
</paths>
```

## 4. Domain Model & Sistem Template Watermark (`MediaCaptureEvent.kt`, `WatermarkTemplate.kt`, `WatermarkTemplateRepository.kt`)

Mencakup struktur data acara tangkapan media, keempat preset bawaan (**Minimalist Corner**, **Modern Pill / Badge**, **Flagship / Studio Footer**, **Cinematic Letterbox**), opsi kustomisasi penuh (`TOP_LEFT`, `TOP_RIGHT`, `BOTTOM_LEFT`, `BOTTOM_RIGHT`, `BOTTOM_BANNER`, opasitas logo `0–100%`, skala logo persentase lebar foto, margin tepi, teks kustom), serta repositori penyimpanan JSON ke `SharedPreferences` & `DataStore`.

```kotlin
package com.automark.quickshare.domain.model

import android.graphics.Color
import android.graphics.Typeface
import android.net.Uri
import android.os.Parcelable
import kotlinx.parcelize.Parcelize
import org.json.JSONObject
import kotlin.math.abs

enum class MediaType {
    IMAGE, VIDEO
}

data class MediaCaptureEvent(
    val sourceUri: Uri,
    val displayName: String,
    val directoryPath: String,
    val legacyFilePath: String?,
    val mimeType: String,
    val sizeBytes: Long,
    val dateAddedSeconds: Long,
    val mediaType: MediaType
)

data class MediaExifInfo(
    val deviceMake: String = "ANDROID",
    val deviceModel: String = "Flagship Camera",
    val focalLength: String = "24mm",
    val aperture: String = "f/1.8",
    val shutterSpeed: String = "1/125s",
    val iso: String = "ISO 100",
    val formattedDateTime: String = "",
    val latitude: Double? = null,
    val longitude: Double? = null,
    val resolvedCityName: String = "Lokasi Luring"
) {
    val exposureSummary: String
        get() = "$focalLength  $aperture  $shutterSpeed  $iso"

    val formattedGpsCoordinates: String
        get() = if (latitude != null && longitude != null) {
            val latHem = if (latitude >= 0) "N" else "S"
            val lonHem = if (longitude >= 0) "E" else "W"
            String.format("%.4f°%s, %.4f°%s", abs(latitude), latHem, abs(longitude), lonHem)
        } else {
            ""
        }
}

enum class PresetStyle(val displayName: String) {
    MINIMALIST_CORNER("Minimalist Corner"),
    MODERN_PILL_BADGE("Modern Pill / Badge"),
    FLAGSHIP_STUDIO_FOOTER("Flagship / Studio Footer"),
    CINEMATIC_LETTERBOX("Cinematic Letterbox"),
    CUSTOM_BUILDER("Custom Template Builder")
}

enum class WatermarkPosition {
    TOP_LEFT,
    TOP_RIGHT,
    BOTTOM_LEFT,
    BOTTOM_RIGHT,
    BOTTOM_BANNER
}

enum class WatermarkFontStyle {
    SANS_BOLD,
    SANS_LIGHT,
    SERIF_ELEGANT,
    MONOSPACE_TECH;

    fun resolveTypeface(): Typeface = when (this) {
        SANS_BOLD -> Typeface.create("sans-serif", Typeface.BOLD)
        SANS_LIGHT -> Typeface.create("sans-serif-light", Typeface.NORMAL)
        SERIF_ELEGANT -> Typeface.create(Typeface.SERIF, Typeface.NORMAL)
        MONOSPACE_TECH -> Typeface.create(Typeface.MONOSPACE, Typeface.BOLD)
    }
}

/**
 * Data Class utama konfigurasi Watermark yang dapat diserialisasi secara ringan ke JSON
 * untuk disimpan di SharedPreferences maupun Jetpack DataStore.
 */
@Parcelize
data class WatermarkTemplate(
    val presetStyle: PresetStyle = PresetStyle.FLAGSHIP_STUDIO_FOOTER,
    val position: WatermarkPosition = WatermarkPosition.BOTTOM_RIGHT,
    /** Opasitas logo PNG transparan (0 - 100%) */
    val logoOpacityPercent: Int = 92,
    /** Skala ukuran logo relatif terhadap lebar foto (misal 0.06f = 6% dari lebar foto) */
    val logoScaleWidthPercent: Float = 0.065f,
    /** Margin/padding horizontal relatif terhadap lebar foto (misal 0.035f = 3.5%) */
    val marginPercentX: Float = 0.035f,
    /** Margin/padding vertikal relatif terhadap tinggi foto (misal 0.035f = 3.5%) */
    val marginPercentY: Float = 0.035f,
    /** Faktor skala ukuran teks relatif (0.5f - 2.0f) */
    val textScaleFactor: Float = 1.0f,
    /** Teks kustom / nama fotografer / nama brand */
    val customText: String = "Shot with AutoMark",
    /** Tampilkan lokasi kota hasil reverse-geocoding offline */
    val showOfflineLocation: Boolean = true,
    /** Tampilkan koordinat GPS mentah */
    val showGpsCoordinates: Boolean = true,
    /** Tampilkan tanggal dan jam pengambilan */
    val showDateTime: Boolean = true,
    /** Warna teks utama (ARGB Int) */
    val textColorArgb: Int = Color.WHITE,
    /** Warna latar kapsul/banner (ARGB Int) */
    val badgeBackgroundColorArgb: Int = Color.argb(175, 16, 16, 20),
    /** Pilihan gaya huruf */
    val fontStyle: WatermarkFontStyle = WatermarkFontStyle.SANS_BOLD,
    /** URI opsional jika pengguna memilih file logo PNG sendiri dari penyimpanan */
    val customLogoUriString: String? = null,
    /** Debounce delay ContentObserver dalam milidetik (500 - 1000 ms) */
    val watcherDebounceMs: Long = 700L
) : Parcelable {

    fun toJsonString(): String {
        return JSONObject().apply {
            put("presetStyle", presetStyle.name)
            put("position", position.name)
            put("logoOpacityPercent", logoOpacityPercent)
            put("logoScaleWidthPercent", logoScaleWidthPercent.toDouble())
            put("marginPercentX", marginPercentX.toDouble())
            put("marginPercentY", marginPercentY.toDouble())
            put("textScaleFactor", textScaleFactor.toDouble())
            put("customText", customText)
            put("showOfflineLocation", showOfflineLocation)
            put("showGpsCoordinates", showGpsCoordinates)
            put("showDateTime", showDateTime)
            put("textColorArgb", textColorArgb)
            put("badgeBackgroundColorArgb", badgeBackgroundColorArgb)
            put("fontStyle", fontStyle.name)
            put("customLogoUriString", customLogoUriString ?: "")
            put("watcherDebounceMs", watcherDebounceMs)
        }.toString()
    }

    companion object {
        fun fromJsonString(jsonString: String?): WatermarkTemplate {
            if (jsonString.isNullOrBlank()) return WatermarkTemplate()
            return runCatching {
                val obj = JSONObject(jsonString)
                WatermarkTemplate(
                    presetStyle = PresetStyle.valueOf(
                        obj.optString("presetStyle", PresetStyle.FLAGSHIP_STUDIO_FOOTER.name)
                    ),
                    position = WatermarkPosition.valueOf(
                        obj.optString("position", WatermarkPosition.BOTTOM_RIGHT.name)
                    ),
                    logoOpacityPercent = obj.optInt("logoOpacityPercent", 92).coerceIn(0, 100),
                    logoScaleWidthPercent = obj.optDouble("logoScaleWidthPercent", 0.065)
                        .toFloat().coerceIn(0.02f, 0.30f),
                    marginPercentX = obj.optDouble("marginPercentX", 0.035)
                        .toFloat().coerceIn(0.0f, 0.25f),
                    marginPercentY = obj.optDouble("marginPercentY", 0.035)
                        .toFloat().coerceIn(0.0f, 0.25f),
                    textScaleFactor = obj.optDouble("textScaleFactor", 1.0)
                        .toFloat().coerceIn(0.5f, 2.0f),
                    customText = obj.optString("customText", "Shot with AutoMark"),
                    showOfflineLocation = obj.optBoolean("showOfflineLocation", true),
                    showGpsCoordinates = obj.optBoolean("showGpsCoordinates", true),
                    showDateTime = obj.optBoolean("showDateTime", true),
                    textColorArgb = obj.optInt("textColorArgb", Color.WHITE),
                    badgeBackgroundColorArgb = obj.optInt(
                        "badgeBackgroundColorArgb",
                        Color.argb(175, 16, 16, 20)
                    ),
                    fontStyle = WatermarkFontStyle.valueOf(
                        obj.optString("fontStyle", WatermarkFontStyle.SANS_BOLD.name)
                    ),
                    customLogoUriString = obj.optString("customLogoUriString", "")
                        .takeIf { it.isNotBlank() },
                    watcherDebounceMs = obj.optLong("watcherDebounceMs", 700L).coerceIn(500L, 1500L)
                )
            }.getOrDefault(WatermarkTemplate())
        }
    }
}
```

### Penyimpanan Template JSON (`WatermarkTemplateRepository.kt`)

```kotlin
package com.automark.quickshare.data.preferences

import android.content.Context
import androidx.datastore.core.DataStore
import androidx.datastore.preferences.core.Preferences
import androidx.datastore.preferences.core.edit
import androidx.datastore.preferences.core.stringPreferencesKey
import androidx.datastore.preferences.preferencesDataStore
import com.automark.quickshare.domain.model.WatermarkTemplate
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.first
import kotlinx.coroutines.flow.map

private val Context.dataStore: DataStore<Preferences> by preferencesDataStore(name = "automark_settings")

class WatermarkTemplateRepository(private val context: Context) {

    private val sharedPrefs = context.applicationContext.getSharedPreferences(
        "automark_quickshare_prefs",
        Context.MODE_PRIVATE
    )

    companion object {
        private val KEY_TEMPLATE_JSON = stringPreferencesKey("watermark_template_json")
        private const val PREF_SYNC_CACHE_KEY = "watermark_template_json_sync"
    }

    val templateFlow: Flow<WatermarkTemplate> = context.applicationContext.dataStore.data.map { prefs ->
        val json = prefs[KEY_TEMPLATE_JSON] ?: sharedPrefs.getString(PREF_SYNC_CACHE_KEY, null)
        WatermarkTemplate.fromJsonString(json)
    }

    suspend fun saveTemplate(template: WatermarkTemplate) {
        val json = template.toJsonString()
        // Simpan sinkron ke SharedPreferences sebagai backup cepat & ke DataStore untuk aliran reaktif
        sharedPrefs.edit().putString(PREF_SYNC_CACHE_KEY, json).apply()
        context.applicationContext.dataStore.edit { prefs ->
            prefs[KEY_TEMPLATE_JSON] = json
        }
    }

    suspend fun getActiveTemplate(): WatermarkTemplate {
        return runCatching { templateFlow.first() }.getOrElse {
            WatermarkTemplate.fromJsonString(sharedPrefs.getString(PREF_SYNC_CACHE_KEY, null))
        }
    }
}
```

## 5. Modul Lokasi Offline (`OfflineGeolocationManager.kt`)

Memenuhi spesifikasi Geolocation Offline secara menyeluruh:
1. Membaca koordinat GPS dari EXIF foto/video terlebih dahulu.
2. Apabila kamera tidak menyematkan koordinat GPS pada file, modul secara otomatis mengambil **lokasi terakhir yang diketahui (*Last Known Location*)** dari [FusedLocationProviderClient.getLastLocation()](https://developer.android.com/develop/sensors-and-location/location/retrieve-current) atau `LocationManager.getLastKnownLocation()` secara pasif tanpa menguras baterai atau membutuhkan internet.
3. Mengonversi koordinat `(latitude, longitude)` menjadi nama kecamatan/kota/kabupaten menggunakan database [SQLiteDatabase](https://developer.android.com/reference/android/database/sqlite/SQLiteDatabase) lokal berindeks *bounding-box* (`min_lat..max_lat`, `min_lon..max_lon`) dan rumus jarak *Haversine*.

```kotlin
package com.automark.quickshare.data.location

import android.Manifest
import android.annotation.SuppressLint
import android.content.Context
import android.content.pm.PackageManager
import android.database.sqlite.SQLiteDatabase
import android.database.sqlite.SQLiteOpenHelper
import android.location.Location
import android.location.LocationManager
import androidx.core.content.ContextCompat
import com.google.android.gms.location.LocationServices
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.tasks.await
import kotlinx.coroutines.withContext
import kotlinx.coroutines.withTimeoutOrNull
import org.json.JSONArray
import kotlin.math.*

data class OfflineCityRecord(
    val cityName: String,
    val districtName: String,
    val provinceName: String,
    val centerLat: Double,
    val centerLon: Double,
    val minLat: Double,
    val maxLat: Double,
    val minLon: Double,
    val maxLon: Double
) {
    val formattedLabel: String
        get() = if (districtName.isNotBlank() && !districtName.equals(cityName, ignoreCase = true)) {
            "$districtName, $cityName"
        } else {
            cityName
        }
}

class OfflineGeolocationManager(context: Context) :
    SQLiteOpenHelper(context.applicationContext, DATABASE_NAME, null, DATABASE_VERSION) {

    private val appContext = context.applicationContext
    private val fusedLocationClient by lazy {
        LocationServices.getFusedLocationProviderClient(appContext)
    }

    companion object {
        private const val DATABASE_NAME = "automark_offline_geo.db"
        private const val DATABASE_VERSION = 1
        private const val TABLE_BOUNDING_BOX = "city_bounding_boxes"
    }

    override fun onCreate(db: SQLiteDatabase) {
        db.execSQL(
            """
            CREATE TABLE $TABLE_BOUNDING_BOX (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                city_name TEXT NOT NULL,
                district_name TEXT NOT NULL,
                province_name TEXT NOT NULL,
                center_lat REAL NOT NULL,
                center_lon REAL NOT NULL,
                min_lat REAL NOT NULL,
                max_lat REAL NOT NULL,
                min_lon REAL NOT NULL,
                max_lon REAL NOT NULL
            )
            """.trimIndent()
        )
        db.execSQL(
            "CREATE INDEX idx_spatial_bbox ON $TABLE_BOUNDING_BOX (min_lat, max_lat, min_lon, max_lon)"
        )
        seedBuiltInCities(db)
    }

    override fun onUpgrade(db: SQLiteDatabase, oldVersion: Int, newVersion: Int) {
        db.execSQL("DROP TABLE IF EXISTS $TABLE_BOUNDING_BOX")
        onCreate(db)
    }

    private fun seedBuiltInCities(db: SQLiteDatabase) {
        val initialCities = listOf(
            OfflineCityRecord("Kab. Malang", "Kepanjen", "Jawa Timur", -8.1303, 112.5725, -8.1900, -8.0700, 112.5100, 112.6300),
            OfflineCityRecord("Kota Malang", "Klojen", "Jawa Timur", -7.9770, 112.6340, -8.0550, -7.9100, 112.5800, 112.6950),
            OfflineCityRecord("Kota Batu", "Batu", "Jawa Timur", -7.8671, 112.5239, -7.9350, -7.8000, 112.4600, 112.5800),
            OfflineCityRecord("Kota Surabaya", "Gubeng", "Jawa Timur", -7.2575, 112.7521, -7.3600, -7.1900, 112.6000, 112.8600),
            OfflineCityRecord("Kab. Sidoarjo", "Sidoarjo", "Jawa Timur", -7.4478, 112.7183, -7.5800, -7.3400, 112.5500, 112.8200),
            OfflineCityRecord("Jakarta Selatan", "Kebayoran Baru", "DKI Jakarta", -6.2442, 106.8006, -6.3600, -6.2000, 106.7300, 106.8700),
            OfflineCityRecord("Jakarta Pusat", "Menteng", "DKI Jakarta", -6.1865, 106.8341, -6.2250, -6.1350, 106.7900, 106.8850),
            OfflineCityRecord("Kota Bandung", "Coblong", "Jawa Barat", -6.8915, 107.6107, -6.9700, -6.8400, 107.5400, 107.7300),
            OfflineCityRecord("Kota Semarang", "Semarang Tengah", "Jawa Tengah", -6.9825, 110.4196, -7.1100, -6.9300, 110.2800, 110.5000),
            OfflineCityRecord("Kota Yogyakarta", "Malioboro", "DI Yogyakarta", -7.7926, 110.3658, -7.8450, -7.7600, 110.3400, 110.4150),
            OfflineCityRecord("Kab. Badung", "Kuta", "Bali", -8.7233, 115.1723, -8.8500, -8.5400, 115.0800, 115.2400),
            OfflineCityRecord("Kota Denpasar", "Denpasar Selatan", "Bali", -8.6705, 115.2126, -8.7500, -8.6000, 115.1700, 115.2750),
            OfflineCityRecord("Kota Medan", "Medan Barat", "Sumatera Utara", 3.5952, 98.6722, 3.4800, 3.7900, 98.5900, 98.7500),
            OfflineCityRecord("Kota Makassar", "Panakkukang", "Sulawesi Selatan", -5.1477, 119.4327, -5.2400, -5.0400, 119.3800, 119.5400)
        )

        db.beginTransaction()
        try {
            val stmt = db.compileStatement(
                """
                INSERT INTO $TABLE_BOUNDING_BOX 
                (city_name, district_name, province_name, center_lat, center_lon, min_lat, max_lat, min_lon, max_lon)
                VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
                """.trimIndent()
            )
            for (item in initialCities) {
                stmt.clearBindings()
                stmt.bindString(1, item.cityName)
                stmt.bindString(2, item.districtName)
                stmt.bindString(3, item.provinceName)
                stmt.bindDouble(4, item.centerLat)
                stmt.bindDouble(5, item.centerLon)
                stmt.bindDouble(6, item.minLat)
                stmt.bindDouble(7, item.maxLat)
                stmt.bindDouble(8, item.minLon)
                stmt.bindDouble(9, item.maxLon)
                stmt.executeInsert()
            }
            db.setTransactionSuccessful()
        } finally {
            db.endTransaction()
        }
    }

    /**
     * Memperkaya database SQLite dari file JSON ringan di assets/offline_cities_bbox.json bila tersedia.
     */
    suspend fun importCustomJsonIfAvailable(assetName: String = "offline_cities_bbox.json") =
        withContext(Dispatchers.IO) {
            runCatching {
                val jsonText = appContext.assets.open(assetName).bufferedReader().use { it.readText() }
                val array = JSONArray(jsonText)
                val db = writableDatabase
                db.beginTransaction()
                try {
                    val stmt = db.compileStatement(
                        """
                        INSERT INTO $TABLE_BOUNDING_BOX
                        (city_name, district_name, province_name, center_lat, center_lon, min_lat, max_lat, min_lon, max_lon)
                        VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
                        """.trimIndent()
                    )
                    for (i in 0 until array.length()) {
                        val obj = array.getJSONObject(i)
                        stmt.clearBindings()
                        stmt.bindString(1, obj.getString("city"))
                        stmt.bindString(2, obj.optString("district", obj.getString("city")))
                        stmt.bindString(3, obj.optString("province", ""))
                        stmt.bindDouble(4, obj.getDouble("centerLat"))
                        stmt.bindDouble(5, obj.getDouble("centerLon"))
                        stmt.bindDouble(6, obj.getDouble("minLat"))
                        stmt.bindDouble(7, obj.getDouble("maxLat"))
                        stmt.bindDouble(8, obj.getDouble("minLon"))
                        stmt.bindDouble(9, obj.getDouble("maxLon"))
                        stmt.executeInsert()
                    }
                    db.setTransactionSuccessful()
                } finally {
                    db.endTransaction()
                }
            }
        }

    /**
     * Mengambil koordinat terbaik:
     * 1. Gunakan koordinat EXIF jika ada.
     * 2. Jika EXIF kosong, gunakan FusedLocationProviderClient.lastLocation atau LocationManager pasif.
     */
    @SuppressLint("MissingPermission")
    suspend fun resolveCoordinatesWithFallback(
        exifLat: Double?,
        exifLon: Double?
    ): Pair<Double, Double>? = withContext(Dispatchers.IO) {
        if (exifLat != null && exifLon != null && (exifLat != 0.0 || exifLon != 0.0)) {
            return@withContext exifLat to exifLon
        }

        val hasFine = ContextCompat.checkSelfPermission(
            appContext,
            Manifest.permission.ACCESS_FINE_LOCATION
        ) == PackageManager.PERMISSION_GRANTED
        val hasCoarse = ContextCompat.checkSelfPermission(
            appContext,
            Manifest.permission.ACCESS_COARSE_LOCATION
        ) == PackageManager.PERMISSION_GRANTED

        if (!hasFine && !hasCoarse) return@withContext null

        // Coba ambil dari FusedLocationProviderClient (cache sistem, 100% hemat baterai & bekerja offline)
        val fusedLocation: Location? = withTimeoutOrNull(600L) {
            runCatching { fusedLocationClient.lastLocation.await() }.getOrNull()
        }
        if (fusedLocation != null) {
            return@withContext fusedLocation.latitude to fusedLocation.longitude
        }

        // Fallback ke LocationManager standar Android (GPS / Network cache pasif)
        return@withContext runCatching {
            val locationManager = appContext.getSystemService(Context.LOCATION_SERVICE) as LocationManager
            val providers = locationManager.getProviders(true)
            var bestLocation: Location? = null
            for (provider in providers) {
                val loc = locationManager.getLastKnownLocation(provider) ?: continue
                if (bestLocation == null || loc.accuracy < bestLocation.accuracy) {
                    bestLocation = loc
                }
            }
            bestLocation?.let { it.latitude to it.longitude }
        }.getOrNull()
    }

    /**
     * Memetakan koordinat GPS ke nama lokasi (Kecamatan, Kota/Kabupaten) secara luring.
     */
    suspend fun reverseGeocodeOffline(lat: Double?, lon: Double?): String =
        withContext(Dispatchers.IO) {
            if (lat == null || lon == null) return@withContext "Indonesia"

            val candidates = mutableListOf<OfflineCityRecord>()
            val db = readableDatabase

            // 1. Cari di dalam bounding-box
            db.rawQuery(
                """
                SELECT city_name, district_name, province_name, center_lat, center_lon, min_lat, max_lat, min_lon, max_lon
                FROM $TABLE_BOUNDING_BOX
                WHERE ? BETWEEN min_lat AND max_lat
                  AND ? BETWEEN min_lon AND max_lon
                """.trimIndent(),
                arrayOf(lat.toString(), lon.toString())
            ).use { cursor ->
                while (cursor.moveToNext()) {
                    candidates.add(
                        OfflineCityRecord(
                            cityName = cursor.getString(0),
                            districtName = cursor.getString(1),
                            provinceName = cursor.getString(2),
                            centerLat = cursor.getDouble(3),
                            centerLon = cursor.getDouble(4),
                            minLat = cursor.getDouble(5),
                            maxLat = cursor.getDouble(6),
                            minLon = cursor.getDouble(7),
                            maxLon = cursor.getDouble(8)
                        )
                    )
                }
            }

            // 2. Jika berada di luar kotak eksak, cari dalam radius +-0.6 derajat (~66 km)
            if (candidates.isEmpty()) {
                val tolerance = 0.6
                db.rawQuery(
                    """
                    SELECT city_name, district_name, province_name, center_lat, center_lon, min_lat, max_lat, min_lon, max_lon
                    FROM $TABLE_BOUNDING_BOX
                    WHERE center_lat BETWEEN ? AND ?
                      AND center_lon BETWEEN ? AND ?
                    """.trimIndent(),
                    arrayOf(
                        (lat - tolerance).toString(),
                        (lat + tolerance).toString(),
                        (lon - tolerance).toString(),
                        (lon + tolerance).toString()
                    )
                ).use { cursor ->
                    while (cursor.moveToNext()) {
                        candidates.add(
                            OfflineCityRecord(
                                cityName = cursor.getString(0),
                                districtName = cursor.getString(1),
                                provinceName = cursor.getString(2),
                                centerLat = cursor.getDouble(3),
                                centerLon = cursor.getDouble(4),
                                minLat = cursor.getDouble(5),
                                maxLat = cursor.getDouble(6),
                                minLon = cursor.getDouble(7),
                                maxLon = cursor.getDouble(8)
                            )
                        )
                    }
                }
            }

            val closest = candidates.minByOrNull { record ->
                haversineKm(lat, lon, record.centerLat, record.centerLon)
            }
            closest?.formattedLabel ?: String.format("%.3f°, %.3f°", lat, lon)
        }

    private fun haversineKm(lat1: Double, lon1: Double, lat2: Double, lon2: Double): Double {
        val r = 6371.0
        val dLat = Math.toRadians(lat2 - lat1)
        val dLon = Math.toRadians(lon2 - lon1)
        val a = sin(dLat / 2).pow(2.0) +
                cos(Math.toRadians(lat1)) * cos(Math.toRadians(lat2)) *
                sin(dLon / 2).pow(2.0)
        return 2 * r * atan2(sqrt(a), sqrt(1 - a))
    }
}
```

## 6. Media Detection & File Watcher (`UniversalMediaContentObserver.kt`)

Dirancang khusus agar tahan terhadap *race condition* dan *IO write lock* pada seluruh merk ponsel Android (Oppo Reno 7/13/14, Samsung, Xiaomi, Vivo):
- **Kompatibilitas API 26–35**: Menggunakan `RELATIVE_PATH` + `IS_PENDING` pada Android 10+ (API 29+) dan `MediaStore.MediaColumns.DATA` pada Android 8.0–9.0 (API 26–28).
- **Debounce Delay (500–1000 ms) & Validasi Kestabilan File**: Menunggu *burst update* `MediaStore` selesai, lalu memvalidasi bahwa ukuran byte file di penyimpanan sudah konstan dan tidak terkunci sebelum memicu proses watermark.
- **Filter Anti-Loop**: Mengabaikan semua file yang memiliki awalan nama `AutoMark_` atau berasal dari direktori `DCIM/AutoMark`.

```kotlin
package com.automark.quickshare.data.media

import android.content.ContentResolver
import android.content.ContentUris
import android.content.Context
import android.database.ContentObserver
import android.net.Uri
import android.os.Build
import android.os.Handler
import android.os.Looper
import android.provider.MediaStore
import android.util.Log
import com.automark.quickshare.domain.model.MediaCaptureEvent
import com.automark.quickshare.domain.model.MediaType
import kotlinx.coroutines.*
import java.io.File
import java.util.Collections
import java.util.LinkedHashMap
import java.util.concurrent.ConcurrentHashMap
import kotlin.math.abs

class UniversalMediaContentObserver(
    context: Context,
    private val coroutineScope: CoroutineScope,
    private val debounceDelayProvider: () -> Long = { 700L },
    private val onCameraMediaValidated: suspend (MediaCaptureEvent) -> Unit
) : ContentObserver(Handler(Looper.getMainLooper())) {

    private val contentResolver: ContentResolver = context.applicationContext.contentResolver
    private val pendingJobs = ConcurrentHashMap<String, Job>()

    // LRU Cache untuk mencegah duplikasi pemrosesan pada URI yang sama
    private val processedUris: MutableSet<String> = Collections.synchronizedSet(
        Collections.newSetFromMap(object : LinkedHashMap<String, Boolean>(400, 0.75f, true) {
            override fun removeEldestEntry(eldest: MutableMap.MutableEntry<String, Boolean>?): Boolean {
                return size > 400
            }
        })
    )

    companion object {
        private const val TAG = "UniversalMediaObserver"
        const val EXPORT_FILE_PREFIX = "AutoMark_"
        const val EXPORT_FOLDER_NAME = "AutoMark"

        // Interval pengecekan ulang bila kamera masih menulis file resolusi tinggi (50MP/108MP/4K Video)
        private val IO_LOCK_RETRY_MS = longArrayOf(400L, 650L, 1000L, 1500L, 2500L)
    }

    fun startObserving() {
        contentResolver.registerContentObserver(
            MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
            true,
            this
        )
        contentResolver.registerContentObserver(
            MediaStore.Video.Media.EXTERNAL_CONTENT_URI,
            true,
            this
        )
        Log.i(TAG, "ContentObserver aktif memantau MediaStore Images & Video (API ${Build.VERSION.SDK_INT}).")
    }

    fun stopObserving() {
        contentResolver.unregisterContentObserver(this)
        pendingJobs.values.forEach { it.cancel() }
        pendingJobs.clear()
    }

    override fun onChange(selfChange: Boolean, uri: Uri?) {
        super.onChange(selfChange, uri)
        if (uri == null) return

        val key = uri.toString()
        if (processedUris.contains(key)) return

        // Reset debounce timer setiap kali kamera memperbarui metadata URI yang sama
        pendingJobs[key]?.cancel()
        pendingJobs[key] = coroutineScope.launch(Dispatchers.IO) {
            try {
                // 1. Debounce delay (500 - 1000 ms) sesuai spesifikasi anti-race condition
                val debounceMs = debounceDelayProvider().coerceIn(500L, 1000L)
                delay(debounceMs)

                // 2. Validasi kelengkapan penulisan buffer & filter anti-loop
                val validEvent = awaitCompleteCameraFile(uri)
                if (validEvent != null && processedUris.add(validEvent.sourceUri.toString())) {
                    processedUris.add(key)
                    Log.d(TAG, "File kamera siap diproses: ${validEvent.displayName} (${validEvent.sizeBytes} bytes)")
                    onCameraMediaValidated(validEvent)
                }
            } catch (e: CancellationException) {
                throw e
            } catch (e: Exception) {
                Log.e(TAG, "Error saat memvalidasi URI media: $uri", e)
            } finally {
                pendingJobs.remove(key)
            }
        }
    }

    private suspend fun awaitCompleteCameraFile(changedUri: Uri): MediaCaptureEvent? {
        val hasSpecificId = parseContentIdOrNull(changedUri) != null

        for (attempt in IO_LOCK_RETRY_MS.indices) {
            val candidate = if (hasSpecificId) {
                inspectSingleUri(changedUri)
            } else {
                inspectMostRecentMedia(changedUri)
            }

            if (candidate != null && verifyFileSizeStabilityAndIoLock(candidate)) {
                return candidate
            }

            delay(IO_LOCK_RETRY_MS[attempt])
        }
        return null
    }

    @Suppress("DEPRECATION")
    private fun inspectSingleUri(uri: Uri): MediaCaptureEvent? {
        val projection = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
            arrayOf(
                MediaStore.MediaColumns._ID,
                MediaStore.MediaColumns.DISPLAY_NAME,
                MediaStore.MediaColumns.MIME_TYPE,
                MediaStore.MediaColumns.SIZE,
                MediaStore.MediaColumns.DATE_ADDED,
                MediaStore.MediaColumns.RELATIVE_PATH,
                MediaStore.MediaColumns.IS_PENDING
            )
        } else {
            arrayOf(
                MediaStore.MediaColumns._ID,
                MediaStore.MediaColumns.DISPLAY_NAME,
                MediaStore.MediaColumns.MIME_TYPE,
                MediaStore.MediaColumns.SIZE,
                MediaStore.MediaColumns.DATE_ADDED,
                MediaStore.MediaColumns.DATA
            )
        }

        contentResolver.query(uri, projection, null, null, null)?.use { cursor ->
            if (!cursor.moveToFirst()) return null

            val displayName = cursor.getString(1) ?: return null
            val mimeType = cursor.getString(2) ?: return null
            val sizeBytes = cursor.getLong(3)
            val dateAddedSec = cursor.getLong(4)

            val directoryPath: String
            val legacyFilePath: String?

            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
                directoryPath = cursor.getString(5) ?: ""
                val isPending = cursor.getInt(6)
                if (isPending != 0) return null // Kamera masih menulis file (Scoped Storage lock)
                legacyFilePath = null
            } else {
                legacyFilePath = cursor.getString(5)
                directoryPath = legacyFilePath?.let { File(it).parent ?: "" } ?: ""
            }

            // Filter 1: Ukuran file harus > 0
            if (sizeBytes <= 0L) return null

            // Filter 2: Anti-Loop — Abaikan file yang diawali "AutoMark_" atau berasal dari folder "DCIM/AutoMark"
            if (displayName.startsWith(EXPORT_FILE_PREFIX, ignoreCase = true) ||
                directoryPath.contains(EXPORT_FOLDER_NAME, ignoreCase = true)
            ) {
                return null
            }

            // Filter 3: Abaikan file temporary kamera (.pending, .trashed, dsb.)
            if (displayName.startsWith(".") || displayName.contains("pending", ignoreCase = true)) {
                return null
            }

            // Filter 4: Pastikan berasal dari folder kamera (DCIM/Camera atau DCIM/100ANDRO, dsb.)
            if (!directoryPath.contains("DCIM", ignoreCase = true) &&
                !directoryPath.contains("Camera", ignoreCase = true)
            ) {
                return null
            }

            // Filter 5: Hanya tangkap media baru (diambil dalam 120 detik terakhir)
            val nowSec = System.currentTimeMillis() / 1000L
            if (abs(nowSec - dateAddedSec) > 120L) return null

            val mediaType = when {
                mimeType.startsWith("image/") -> MediaType.IMAGE
                mimeType.startsWith("video/") -> MediaType.VIDEO
                else -> return null
            }

            return MediaCaptureEvent(
                sourceUri = uri,
                displayName = displayName,
                directoryPath = directoryPath,
                legacyFilePath = legacyFilePath,
                mimeType = mimeType,
                sizeBytes = sizeBytes,
                dateAddedSeconds = dateAddedSec,
                mediaType = mediaType
            )
        }
        return null
    }

    private fun inspectMostRecentMedia(tableUri: Uri): MediaCaptureEvent? {
        val projection = arrayOf(MediaStore.MediaColumns._ID)
        val sortOrder = "${MediaStore.MediaColumns.DATE_ADDED} DESC"
        contentResolver.query(tableUri, projection, null, null, sortOrder)?.use { cursor ->
            if (cursor.moveToFirst()) {
                val id = cursor.getLong(0)
                val rowUri = ContentUris.withAppendedId(tableUri, id)
                if (!processedUris.contains(rowUri.toString())) {
                    return inspectSingleUri(rowUri)
                }
            }
        }
        return null
    }

    /**
     * Memastikan file sudah tidak sedang ditulis oleh aplikasi kamera bawaan:
     * Mengukur panjang byte file dua kali dengan jeda 250ms dan memastikan ukurannya konstan (> 0 byte)
     * serta stream dapat dibuka untuk pembacaan.
     */
    private suspend fun verifyFileSizeStabilityAndIoLock(event: MediaCaptureEvent): Boolean {
        return try {
            val size1 = readCurrentByteLength(event)
            if (size1 <= 0L) return false
            delay(250L)
            val size2 = readCurrentByteLength(event)
            size1 == size2 && size2 > 0L
        } catch (_: Exception) {
            false
        }
    }

    private fun readCurrentByteLength(event: MediaCaptureEvent): Long {
        if (Build.VERSION.SDK_INT < Build.VERSION_CODES.Q && !event.legacyFilePath.isNullOrBlank()) {
            val f = File(event.legacyFilePath)
            if (!f.exists() || !f.canRead()) return -1L
            return f.length()
        }
        return contentResolver.openAssetFileDescriptor(event.sourceUri, "r")?.use { afd ->
            afd.length
        } ?: -1L
    }

    private fun parseContentIdOrNull(uri: Uri): Long? = try {
        ContentUris.parseId(uri)
    } catch (_: Exception) {
        null
    }
}
```

## 7. Watermark Rendering Engine (`WatermarkRenderer.kt`)

Mendukung pemrosesan foto beresolusi tinggi (12 MP, 50 MP, 64 MP, hingga 108 MP) dan video 30/60 FPS:
- **Modul Foto (`Bitmap.Config.ARGB_8888`)**: Seluruh koordinat, ukuran font, padding kapsul, dan skala logo dihitung menggunakan persentase proporsional terhadap dimensi asli foto, lalu metadata EXIF asli diduplikasi menggunakan [ExifInterface](https://developer.android.com/reference/androidx/exifinterface/media/ExifInterface).
- **Modul Video (Hardware-Accelerated)**: Menggunakan [AndroidX Media3 Transformer](https://developer.android.com/reference/androidx/media3/transformer/Transformer) dengan `BitmapOverlay` dan `StaticOverlaySettings` (`OverlayEffect`) agar komposisi dilakukan di GPU tanpa menurunkan resolusi maupun *framerate* asli.
- **Penyimpanan Lintas SDK (API 26–35)**: Menyimpan ke `DCIM/AutoMark` dengan nama file `AutoMark_<timestamp>_<nama_asli>`. Pada Android 10+ menggunakan `RELATIVE_PATH` + `IS_PENDING`, sedangkan pada Android 8.0–9.0 menggunakan `File` langsung + [MediaScannerConnection.scanFile()](https://developer.android.com/reference/android/media/MediaScannerConnection) dan `FileProvider`.

```kotlin
package com.automark.quickshare.data.media

import android.content.ContentValues
import android.content.Context
import android.graphics.*
import android.media.MediaMetadataRetriever
import android.media.MediaScannerConnection
import android.net.Uri
import android.os.Build
import android.os.Environment
import android.os.Looper
import android.provider.MediaStore
import android.text.TextPaint
import androidx.annotation.OptIn
import androidx.core.content.FileProvider
import androidx.exifinterface.media.ExifInterface
import androidx.media3.common.Effect
import androidx.media3.common.MediaItem
import androidx.media3.common.util.UnstableApi
import androidx.media3.effect.BitmapOverlay
import androidx.media3.effect.OverlayEffect
import androidx.media3.effect.StaticOverlaySettings
import androidx.media3.transformer.Composition
import androidx.media3.transformer.EditedMediaItem
import androidx.media3.transformer.Effects
import androidx.media3.transformer.ExportException
import androidx.media3.transformer.ExportResult
import androidx.media3.transformer.Transformer
import com.automark.quickshare.data.location.OfflineGeolocationManager
import com.automark.quickshare.domain.model.*
import com.google.common.collect.ImmutableList
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.suspendCancellableCoroutine
import kotlinx.coroutines.withContext
import java.io.File
import java.io.FileOutputStream
import java.text.SimpleDateFormat
import java.util.Date
import java.util.Locale
import kotlin.coroutines.resume
import kotlin.coroutines.resumeWithException
import kotlin.math.max
import kotlin.math.min
import kotlin.math.roundToInt

class WatermarkRenderer(
    private val context: Context,
    private val geolocationManager: OfflineGeolocationManager
) {

    private val appContext = context.applicationContext
    private val contentResolver = appContext.contentResolver

    companion object {
        const val OUTPUT_SUBFOLDER = "DCIM/AutoMark"

        private val EXIF_TAGS_TO_COPY = arrayOf(
            ExifInterface.TAG_MAKE,
            ExifInterface.TAG_MODEL,
            ExifInterface.TAG_SOFTWARE,
            ExifInterface.TAG_DATETIME,
            ExifInterface.TAG_DATETIME_ORIGINAL,
            ExifInterface.TAG_DATETIME_DIGITIZED,
            ExifInterface.TAG_SUBSEC_TIME,
            ExifInterface.TAG_SUBSEC_TIME_ORIGINAL,
            ExifInterface.TAG_EXPOSURE_TIME,
            ExifInterface.TAG_F_NUMBER,
            ExifInterface.TAG_APERTURE_VALUE,
            ExifInterface.TAG_ISO_SPEED_RATINGS,
            ExifInterface.TAG_PHOTOGRAPHIC_SENSITIVITY,
            ExifInterface.TAG_FOCAL_LENGTH,
            ExifInterface.TAG_FOCAL_LENGTH_IN_35MM_FILM,
            ExifInterface.TAG_WHITE_BALANCE,
            ExifInterface.TAG_FLASH,
            ExifInterface.TAG_COLOR_SPACE,
            ExifInterface.TAG_GPS_LATITUDE,
            ExifInterface.TAG_GPS_LATITUDE_REF,
            ExifInterface.TAG_GPS_LONGITUDE,
            ExifInterface.TAG_GPS_LONGITUDE_REF,
            ExifInterface.TAG_GPS_ALTITUDE,
            ExifInterface.TAG_GPS_ALTITUDE_REF,
            ExifInterface.TAG_GPS_TIMESTAMP,
            ExifInterface.TAG_GPS_DATESTAMP
        )
    }

    /**
     * Modul A: Pemrosesan Foto dengan ARGB_8888, skala proporsional penuh (12MP - 108MP),
     * dan duplikasi metadata EXIF asli.
     */
    suspend fun processPhoto(
        event: MediaCaptureEvent,
        template: WatermarkTemplate,
        logoBitmap: Bitmap?
    ): Uri = withContext(Dispatchers.IO) {
        val accessibleUri = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
            runCatching { MediaStore.setRequireOriginal(event.sourceUri) }.getOrDefault(event.sourceUri)
        } else {
            event.sourceUri
        }

        // 1. Ekstraksi metadata EXIF & Lokasi Offline (EXIF GPS -> Fallback FusedLocationProviderClient)
        val exifInfo = extractPhotoMetadataAndLocation(accessibleUri)

        // 2. Decode dengan Bitmap.Config.ARGB_8888 untuk menjaga akurasi profil warna asli
        val options = BitmapFactory.Options().apply {
            inPreferredConfig = Bitmap.Config.ARGB_8888
            inMutable = true
        }

        val decodedBitmap = contentResolver.openInputStream(accessibleUri)?.use { stream ->
            BitmapFactory.decodeStream(stream, null, options)
        } ?: throw IllegalStateException("Tidak dapat mendekode foto: ${event.sourceUri}")

        // 3. Normalisasi orientasi berdasarkan tag EXIF Orientation
        val rotationDegrees = readExifRotation(accessibleUri)
        val orientedPhoto = if (rotationDegrees != 0) {
            val matrix = Matrix().apply { postRotate(rotationDegrees.toFloat()) }
            val rotated = Bitmap.createBitmap(
                decodedBitmap, 0, 0, decodedBitmap.width, decodedBitmap.height, matrix, true
            )
            if (rotated != decodedBitmap) decodedBitmap.recycle()
            rotated
        } else {
            decodedBitmap
        }

        // 4. Apabila menggunakan preset FLAGSHIP_STUDIO_FOOTER, tambahkan tinggi kanvas di bawah foto
        val isFlagshipFooter = template.presetStyle == PresetStyle.FLAGSHIP_STUDIO_FOOTER
        val extraFooterPx = if (isFlagshipFooter) {
            (orientedPhoto.height * 0.11f * template.textScaleFactor).roundToInt()
        } else {
            0
        }

        val targetCanvasBitmap = if (extraFooterPx > 0 || !orientedPhoto.isMutable) {
            val combined = Bitmap.createBitmap(
                orientedPhoto.width,
                orientedPhoto.height + extraFooterPx,
                Bitmap.Config.ARGB_8888
            )
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
                orientedPhoto.colorSpace?.let { combined.setColorSpace(it) }
            }
            val c = Canvas(combined)
            c.drawBitmap(orientedPhoto, 0f, 0f, null)
            orientedPhoto.recycle()
            combined
        } else {
            orientedPhoto
        }

        val photoHeight = targetCanvasBitmap.height - extraFooterPx
        val canvas = Canvas(targetCanvasBitmap)

        // 5. Gambar Watermark sesuai Preset atau Custom Template Builder
        drawWatermarkOnCanvas(
            canvas = canvas,
            canvasWidth = targetCanvasBitmap.width,
            canvasHeight = targetCanvasBitmap.height,
            photoContentHeight = photoHeight,
            extraFooterHeight = extraFooterPx,
            exif = exifInfo,
            template = template,
            logoBitmap = logoBitmap
        )

        // 6. Simpan ke DCIM/AutoMark dengan kualitas JPEG 100% + Duplikasi EXIF
        val exportName = "${UniversalMediaContentObserver.EXPORT_FILE_PREFIX}${System.currentTimeMillis()}_${event.displayName}"
        val savedUri = saveWatermarkedBitmapCrossVersion(
            bitmap = targetCanvasBitmap,
            sourceUri = accessibleUri,
            displayName = exportName,
            mimeType = "image/jpeg"
        )
        targetCanvasBitmap.recycle()
        savedUri
    }

    /**
     * Modul B: Pemrosesan Video Hardware-Accelerated menggunakan AndroidX Media3 Transformer
     * dengan BitmapOverlay / StaticOverlaySettings tanpa menurunkan resolusi maupun framerate (30/60 FPS).
     */
    @OptIn(UnstableApi::class)
    suspend fun processVideo(
        event: MediaCaptureEvent,
        template: WatermarkTemplate,
        logoBitmap: Bitmap?
    ): Uri = withContext(Dispatchers.IO) {
        val accessibleUri = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
            runCatching { MediaStore.setRequireOriginal(event.sourceUri) }.getOrDefault(event.sourceUri)
        } else {
            event.sourceUri
        }

        // 1. Ekstraksi dimensi video & lokasi offline
        val (videoSize, exifInfo) = extractVideoMetadataAndLocation(accessibleUri)

        // 2. Render watermark ke dalam Bitmap transparan ARGB_8888 sesuai dimensi video
        val overlayBitmap = createTransparentVideoOverlay(
            width = videoSize.width,
            height = videoSize.height,
            exif = exifInfo,
            template = template,
            logoBitmap = logoBitmap
        )

        // 3. Konfigurasi OverlaySettings & BitmapOverlay untuk Media3 Transformer
        val overlaySettings = StaticOverlaySettings.Builder()
            .setAlphaScale(1.0f)
            .build()
        val bitmapOverlay = BitmapOverlay.createStaticBitmapOverlay(overlayBitmap, overlaySettings)
        val overlayEffect = OverlayEffect(ImmutableList.of(bitmapOverlay))

        val tempVideoFile = File(appContext.cacheDir, "automark_vid_${System.currentTimeMillis()}.mp4")

        try {
            // 4. Jalankan transkoding GPU Media3 Transformer
            executeMedia3Transformation(
                inputUri = accessibleUri,
                videoEffects = listOf(overlayEffect),
                outputFilePath = tempVideoFile.absolutePath
            )

            // 5. Simpan hasil video ke DCIM/AutoMark (mendukung API 26-35)
            val exportName = "${UniversalMediaContentObserver.EXPORT_FILE_PREFIX}${System.currentTimeMillis()}_${event.displayName}"
            saveWatermarkedVideoCrossVersion(
                tempFile = tempVideoFile,
                displayName = exportName,
                mimeType = event.mimeType.ifBlank { "video/mp4" }
            )
        } finally {
            overlayBitmap.recycle()
            if (tempVideoFile.exists()) tempVideoFile.delete()
        }
    }

    @OptIn(UnstableApi::class)
    private suspend fun executeMedia3Transformation(
        inputUri: Uri,
        videoEffects: List<Effect>,
        outputFilePath: String
    ) = withContext(Dispatchers.Main) {
        suspendCancellableCoroutine<ExportResult> { cont ->
            val listener = object : Transformer.Listener {
                override fun onCompleted(composition: Composition, exportResult: ExportResult) {
                    if (cont.isActive) cont.resume(exportResult)
                }

                override fun onError(
                    composition: Composition,
                    exportResult: ExportResult,
                    exportException: ExportException
                ) {
                    if (cont.isActive) cont.resumeWithException(exportException)
                }
            }

            val transformer = Transformer.Builder(appContext)
                .setLooper(Looper.getMainLooper())
                .addListener(listener)
                .build()

            val editedMediaItem = EditedMediaItem.Builder(MediaItem.fromUri(inputUri))
                .setEffects(Effects(emptyList(), videoEffects))
                .build()

            transformer.start(editedMediaItem, outputFilePath)

            cont.invokeOnCancellation {
                transformer.cancel()
            }
        }
    }

    private fun createTransparentVideoOverlay(
        width: Int,
        height: Int,
        exif: MediaExifInfo,
        template: WatermarkTemplate,
        logoBitmap: Bitmap?
    ): Bitmap {
        val overlay = Bitmap.createBitmap(width, height, Bitmap.Config.ARGB_8888)
        val canvas = Canvas(overlay)
        canvas.drawColor(Color.TRANSPARENT, PorterDuff.Mode.CLEAR)

        // Pada video, Flagship Footer digambar sebagai banner bawah di dalam bingkai video
        drawWatermarkOnCanvas(
            canvas = canvas,
            canvasWidth = width,
            canvasHeight = height,
            photoContentHeight = height,
            extraFooterHeight = 0,
            exif = exif,
            template = template,
            logoBitmap = logoBitmap
        )
        return overlay
    }

    /**
     * Mesin penggambaran proporsional untuk 4 Preset Bawaan + Custom Template Builder.
     */
    private fun drawWatermarkOnCanvas(
        canvas: Canvas,
        canvasWidth: Int,
        canvasHeight: Int,
        photoContentHeight: Int,
        extraFooterHeight: Int,
        exif: MediaExifInfo,
        template: WatermarkTemplate,
        logoBitmap: Bitmap?
    ) {
        val minSide = min(canvasWidth, photoContentHeight).toFloat()
        val textScale = template.textScaleFactor.coerceIn(0.5f, 2.0f)

        when (template.presetStyle) {
            PresetStyle.MINIMALIST_CORNER -> {
                renderMinimalistCorner(canvas, canvasWidth, photoContentHeight, minSide, textScale, exif, template, logoBitmap)
            }
            PresetStyle.MODERN_PILL_BADGE -> {
                renderModernPillBadge(canvas, canvasWidth, photoContentHeight, minSide, textScale, exif, template, logoBitmap)
            }
            PresetStyle.FLAGSHIP_STUDIO_FOOTER -> {
                renderFlagshipStudioFooter(canvas, canvasWidth, photoContentHeight, extraFooterHeight, textScale, exif, template, logoBitmap)
            }
            PresetStyle.CINEMATIC_LETTERBOX -> {
                renderCinematicLetterbox(canvas, canvasWidth, photoContentHeight, textScale, exif, template, logoBitmap)
            }
            PresetStyle.CUSTOM_BUILDER -> {
                renderCustomTemplateBuilder(canvas, canvasWidth, photoContentHeight, minSide, textScale, exif, template, logoBitmap)
            }
        }
    }

    /**
     * Preset 1: "Minimalist Corner" — Logo kecil + stempel tanggal & lokasi di pojok bawah.
     */
    private fun renderMinimalistCorner(
        canvas: Canvas,
        width: Int,
        height: Int,
        minSide: Float,
        textScale: Float,
        exif: MediaExifInfo,
        template: WatermarkTemplate,
        logoBitmap: Bitmap?
    ) {
        val marginX = width * template.marginPercentX
        val marginY = height * template.marginPercentY
        val titleSize = minSide * 0.027f * textScale
        val subSize = minSide * 0.020f * textScale

        val titlePaint = TextPaint(Paint.ANTI_ALIAS_FLAG).apply {
            color = template.textColorArgb
            textSize = titleSize
            typeface = Typeface.create("sans-serif-medium", Typeface.BOLD)
            setShadowLayer(titleSize * 0.25f, 0f, titleSize * 0.08f, Color.argb(170, 0, 0, 0))
        }
        val datePaint = TextPaint(Paint.ANTI_ALIAS_FLAG).apply {
            color = template.textColorArgb
            alpha = 225
            textSize = subSize
            typeface = Typeface.create("sans-serif", Typeface.NORMAL)
            setShadowLayer(subSize * 0.25f, 0f, subSize * 0.08f, Color.argb(170, 0, 0, 0))
        }

        val topLine = buildString {
            append(template.customText.ifBlank { exif.deviceModel })
            if (template.showOfflineLocation) append(" • ${exif.resolvedCityName}")
        }
        val bottomLine = if (template.showDateTime) exif.formattedDateTime else exif.formattedGpsCoordinates

        val textBlockW = max(titlePaint.measureText(topLine), datePaint.measureText(bottomLine))
        val logoW = if (logoBitmap != null) width * template.logoScaleWidthPercent else 0f
        val logoH = if (logoBitmap != null) {
            logoW * (logoBitmap.height.toFloat() / logoBitmap.width.coerceAtLeast(1))
        } else 0f
        val gap = minSide * 0.014f

        val totalWidth = (if (logoBitmap != null) logoW + gap else 0f) + textBlockW
        val startX = width - marginX - totalWidth
        val bottomY = height - marginY

        if (logoBitmap != null) {
            val logoPaint = Paint(Paint.ANTI_ALIAS_FLAG or Paint.FILTER_BITMAP_FLAG).apply {
                alpha = (template.logoOpacityPercent.coerceIn(0, 100) * 255) / 100
            }
            val dst = RectF(startX, bottomY - logoH, startX + logoW, bottomY)
            canvas.drawBitmap(logoBitmap, null, dst, logoPaint)
        }

        val textX = startX + (if (logoBitmap != null) logoW + gap else 0f)
        canvas.drawText(topLine, textX, bottomY - subSize * 1.35f, titlePaint)
        canvas.drawText(bottomLine, textX, bottomY - subSize * 0.15f, datePaint)
    }

    /**
     * Preset 2: "Modern Pill / Badge" — Kapsul hitam transparan memuat logo, pin lokasi, dan teks bold.
     */
    private fun renderModernPillBadge(
        canvas: Canvas,
        width: Int,
        height: Int,
        minSide: Float,
        textScale: Float,
        exif: MediaExifInfo,
        template: WatermarkTemplate,
        logoBitmap: Bitmap?
    ) {
        val titleSize = minSide * 0.025f * textScale
        val subSize = minSide * 0.019f * textScale
        val padH = minSide * 0.026f * textScale
        val padV = minSide * 0.016f * textScale

        val boldPaint = TextPaint(Paint.ANTI_ALIAS_FLAG).apply {
            color = template.textColorArgb
            textSize = titleSize
            typeface = Typeface.create("sans-serif", Typeface.BOLD)
        }
        val subPaint = TextPaint(Paint.ANTI_ALIAS_FLAG).apply {
            color = template.textColorArgb
            alpha = 215
            textSize = subSize
            typeface = Typeface.create("sans-serif", Typeface.NORMAL)
        }

        val line1 = "📍 ${exif.resolvedCityName}"
        val line2 = buildString {
            append(template.customText.ifBlank { exif.deviceModel })
            if (template.showDateTime) append(" • ${exif.formattedDateTime}")
        }

        val maxTextW = max(boldPaint.measureText(line1), subPaint.measureText(line2))
        val logoW = if (logoBitmap != null) width * template.logoScaleWidthPercent else 0f
        val logoH = if (logoBitmap != null) {
            logoW * (logoBitmap.height.toFloat() / logoBitmap.width.coerceAtLeast(1))
        } else 0f
        val gap = minSide * 0.015f * textScale

        val pillW = padH * 2 + maxTextW + (if (logoBitmap != null) logoW + gap else 0f)
        val pillH = max(logoH + padV * 1.6f, padV * 2 + titleSize + subSize * 1.35f)

        val pillRect = computePositionedRect(
            canvasWidth = width,
            canvasHeight = height,
            boxWidth = pillW,
            boxHeight = pillH,
            position = template.position,
            marginPercentX = template.marginPercentX,
            marginPercentY = template.marginPercentY
        )

        val bgPaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
            color = template.badgeBackgroundColorArgb
            style = Paint.Style.FILL
        }
        val strokePaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
            color = Color.argb(75, 255, 255, 255)
            style = Paint.Style.STROKE
            strokeWidth = max(2f, minSide * 0.0022f)
        }

        val radius = pillH / 2f
        canvas.drawRoundRect(pillRect, radius, radius, bgPaint)
        canvas.drawRoundRect(pillRect, radius, radius, strokePaint)

        var cursorX = pillRect.left + padH
        if (logoBitmap != null) {
            val logoTop = pillRect.centerY() - logoH / 2f
            val dst = RectF(cursorX, logoTop, cursorX + logoW, logoTop + logoH)
            val logoPaint = Paint(Paint.ANTI_ALIAS_FLAG or Paint.FILTER_BITMAP_FLAG).apply {
                alpha = (template.logoOpacityPercent.coerceIn(0, 100) * 255) / 100
            }
            canvas.drawBitmap(logoBitmap, null, dst, logoPaint)
            cursorX += logoW + gap
        }

        val y1 = pillRect.centerY() - (subSize * 0.25f)
        val y2 = y1 + subSize * 1.28f
        canvas.drawText(line1, cursorX, y1, boldPaint)
        canvas.drawText(line2, cursorX, y2, subPaint)
    }

    /**
     * Preset 3: "Flagship / Studio Footer" — Banner putih elegan di bagian bawah (gaya Leica/Hasselblad).
     */
    private fun renderFlagshipStudioFooter(
        canvas: Canvas,
        width: Int,
        photoHeight: Int,
        extraFooterHeight: Int,
        textScale: Float,
        exif: MediaExifInfo,
        template: WatermarkTemplate,
        logoBitmap: Bitmap?
    ) {
        val footerH = if (extraFooterHeight > 0) {
            extraFooterHeight.toFloat()
        } else {
            photoHeight * 0.11f * textScale
        }
        val topY = if (extraFooterHeight > 0) photoHeight.toFloat() else (photoHeight - footerH)
        val bottomY = topY + footerH
        val centerY = (topY + bottomY) / 2f

        val bgPaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
            color = Color.WHITE
            style = Paint.Style.FILL
        }
        canvas.drawRect(0f, topY, width.toFloat(), bottomY, bgPaint)

        val padX = width * max(template.marginPercentX, 0.04f)
        val modelPaint = TextPaint(Paint.ANTI_ALIAS_FLAG).apply {
            color = Color.rgb(18, 18, 22)
            textSize = footerH * 0.26f
            typeface = Typeface.create("sans-serif", Typeface.BOLD)
        }
        val subPaint = TextPaint(Paint.ANTI_ALIAS_FLAG).apply {
            color = Color.rgb(105, 108, 114)
            textSize = footerH * 0.175f
            typeface = Typeface.create("sans-serif", Typeface.NORMAL)
        }
        val specPaint = TextPaint(Paint.ANTI_ALIAS_FLAG).apply {
            color = Color.rgb(22, 22, 26)
            textSize = footerH * 0.21f
            typeface = Typeface.create("sans-serif-medium", Typeface.BOLD)
        }

        // Sisi Kiri: Nama Perangkat / Teks Kustom + Lokasi Offline & Koordinat
        val leftTitle = exif.deviceModel.ifBlank { template.customText }
        val leftSubtitle = buildString {
            if (template.showOfflineLocation) append(exif.resolvedCityName)
            if (template.showGpsCoordinates && exif.formattedGpsCoordinates.isNotEmpty()) {
                if (isNotEmpty()) append("  •  ")
                append(exif.formattedGpsCoordinates)
            }
        }
        canvas.drawText(leftTitle, padX, centerY - footerH * 0.04f, modelPaint)
        canvas.drawText(leftSubtitle, padX, centerY + footerH * 0.22f, subPaint)

        // Sisi Kanan: Logo + Garis Vertikal + Parameter EXIF & Tanggal
        val rightTop = exif.exposureSummary
        val rightBottom = if (template.showDateTime) exif.formattedDateTime else template.customText
        val rightBlockW = max(specPaint.measureText(rightTop), subPaint.measureText(rightBottom))
        val rightStartX = width - padX - rightBlockW

        canvas.drawText(rightTop, rightStartX, centerY - footerH * 0.04f, specPaint)
        canvas.drawText(rightBottom, rightStartX, centerY + footerH * 0.22f, subPaint)

        val dividerX = rightStartX - (footerH * 0.28f)
        val dividerPaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
            color = Color.rgb(210, 212, 216)
            strokeWidth = max(2f, footerH * 0.02f)
        }
        canvas.drawLine(dividerX, centerY - footerH * 0.26f, dividerX, centerY + footerH * 0.26f, dividerPaint)

        if (logoBitmap != null) {
            val logoH = footerH * 0.48f
            val logoW = logoH * (logoBitmap.width.toFloat() / logoBitmap.height.coerceAtLeast(1))
            val logoLeft = dividerX - (footerH * 0.28f) - logoW
            val logoPaint = Paint(Paint.ANTI_ALIAS_FLAG or Paint.FILTER_BITMAP_FLAG).apply {
                alpha = (template.logoOpacityPercent.coerceIn(0, 100) * 255) / 100
            }
            val dst = RectF(logoLeft, centerY - logoH / 2f, logoLeft + logoW, centerY + logoH / 2f)
            canvas.drawBitmap(logoBitmap, null, dst, logoPaint)
        }
    }

    /**
     * Preset 4: "Cinematic Letterbox" — Garis bar hitam atas-bawah dengan tipografi bersih.
     */
    private fun renderCinematicLetterbox(
        canvas: Canvas,
        width: Int,
        height: Int,
        textScale: Float,
        exif: MediaExifInfo,
        template: WatermarkTemplate,
        logoBitmap: Bitmap?
    ) {
        val barH = (height * 0.075f * textScale).coerceAtLeast(56f)
        val barPaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
            color = Color.argb(238, 10, 10, 12)
            style = Paint.Style.FILL
        }

        // Bar hitam atas dan bawah
        canvas.drawRect(0f, 0f, width.toFloat(), barH, barPaint)
        canvas.drawRect(0f, height - barH, width.toFloat(), height.toFloat(), barPaint)

        val padX = width * max(template.marginPercentX, 0.04f)
        val bottomCenterY = height - (barH / 2f)

        val leftPaint = TextPaint(Paint.ANTI_ALIAS_FLAG).apply {
            color = Color.WHITE
            textSize = barH * 0.27f
            typeface = Typeface.create("sans-serif-condensed", Typeface.BOLD)
            letterSpacing = 0.10f
        }
        val rightPaint = TextPaint(Paint.ANTI_ALIAS_FLAG).apply {
            color = Color.argb(210, 225, 225, 230)
            textSize = barH * 0.22f
            typeface = Typeface.create(Typeface.MONOSPACE, Typeface.NORMAL)
        }

        var cursorX = padX
        if (logoBitmap != null) {
            val logoH = barH * 0.52f
            val logoW = logoH * (logoBitmap.width.toFloat() / logoBitmap.height.coerceAtLeast(1))
            val logoPaint = Paint(Paint.ANTI_ALIAS_FLAG or Paint.FILTER_BITMAP_FLAG).apply {
                alpha = (template.logoOpacityPercent.coerceIn(0, 100) * 255) / 100
            }
            val dst = RectF(cursorX, bottomCenterY - logoH / 2f, cursorX + logoW, bottomCenterY + logoH / 2f)
            canvas.drawBitmap(logoBitmap, null, dst, logoPaint)
            cursorX += logoW + (barH * 0.35f)
        }

        val leftLabel = buildString {
            append(template.customText.ifBlank { exif.deviceModel }.uppercase(Locale.ROOT))
            if (template.showOfflineLocation) append("  |  ${exif.resolvedCityName.uppercase(Locale.ROOT)}")
        }
        canvas.drawText(leftLabel, cursorX, bottomCenterY + leftPaint.textSize * 0.35f, leftPaint)

        val rightLabel = buildString {
            if (template.showGpsCoordinates && exif.formattedGpsCoordinates.isNotEmpty()) {
                append(exif.formattedGpsCoordinates)
                append("   ")
            }
            if (template.showDateTime) append(exif.formattedDateTime)
        }
        val rightW = rightPaint.measureText(rightLabel)
        canvas.drawText(rightLabel, width - padX - rightW, bottomCenterY + rightPaint.textSize * 0.35f, rightPaint)
    }

    /**
     * Custom Template Builder: Mendukung posisi TOP_LEFT, TOP_RIGHT, BOTTOM_LEFT, BOTTOM_RIGHT, BOTTOM_BANNER,
     * skala ukuran logo berbasis persentase lebar foto (logoScaleWidthPercent), opasitas logo (0-100%),
     * dan margin tepi persentase.
     */
    private fun renderCustomTemplateBuilder(
        canvas: Canvas,
        width: Int,
        height: Int,
        minSide: Float,
        textScale: Float,
        exif: MediaExifInfo,
        template: WatermarkTemplate,
        logoBitmap: Bitmap?
    ) {
        if (template.position == WatermarkPosition.BOTTOM_BANNER) {
            renderCinematicLetterbox(canvas, width, height, textScale, exif, template, logoBitmap)
            return
        }

        val customTypeface = template.fontStyle.resolveTypeface()
        val titleSize = minSide * 0.026f * textScale
        val subSize = minSide * 0.019f * textScale

        val mainPaint = TextPaint(Paint.ANTI_ALIAS_FLAG).apply {
            color = template.textColorArgb
            textSize = titleSize
            typeface = customTypeface
            setShadowLayer(titleSize * 0.20f, 0f, titleSize * 0.08f, Color.argb(175, 0, 0, 0))
        }
        val subPaint = TextPaint(Paint.ANTI_ALIAS_FLAG).apply {
            color = template.textColorArgb
            alpha = 225
            textSize = subSize
            typeface = customTypeface
            setShadowLayer(subSize * 0.20f, 0f, subSize * 0.08f, Color.argb(175, 0, 0, 0))
        }

        val line1 = template.customText.ifBlank { exif.deviceModel }
        val line2 = buildString {
            if (template.showOfflineLocation) append("📍 ${exif.resolvedCityName}")
            if (template.showGpsCoordinates && exif.formattedGpsCoordinates.isNotEmpty()) {
                if (isNotEmpty()) append(" • ")
                append(exif.formattedGpsCoordinates)
            }
        }.ifBlank { exif.exposureSummary }
        val line3 = if (template.showDateTime) exif.formattedDateTime else exif.exposureSummary

        val textMaxW = maxOf(
            mainPaint.measureText(line1),
            subPaint.measureText(line2),
            subPaint.measureText(line3)
        )

        // Skala ukuran logo dihitung langsung dari persentase lebar foto (logoScaleWidthPercent)
        val logoW = if (logoBitmap != null) width * template.logoScaleWidthPercent else 0f
        val logoH = if (logoBitmap != null) {
            logoW * (logoBitmap.height.toFloat() / logoBitmap.width.coerceAtLeast(1))
        } else 0f

        val pad = minSide * 0.020f * textScale
        val gap = minSide * 0.015f * textScale

        val textStackH = titleSize + (subSize * 2.4f)
        val boxW = pad * 2 + textMaxW + (if (logoBitmap != null) logoW + gap else 0f)
        val boxH = pad * 2 + max(logoH, textStackH)

        val rect = computePositionedRect(
            canvasWidth = width,
            canvasHeight = height,
            boxWidth = boxW,
            boxHeight = boxH,
            position = template.position,
            marginPercentX = template.marginPercentX,
            marginPercentY = template.marginPercentY
        )

        val bgPaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
            color = template.badgeBackgroundColorArgb
            style = Paint.Style.FILL
        }
        val cornerRadius = minSide * 0.018f * textScale
        canvas.drawRoundRect(rect, cornerRadius, cornerRadius, bgPaint)

        var textStartX = rect.left + pad
        if (logoBitmap != null) {
            val logoTop = rect.centerY() - logoH / 2f
            val dst = RectF(textStartX, logoTop, textStartX + logoW, logoTop + logoH)
            val logoPaint = Paint(Paint.ANTI_ALIAS_FLAG or Paint.FILTER_BITMAP_FLAG).apply {
                alpha = (template.logoOpacityPercent.coerceIn(0, 100) * 255) / 100
            }
            canvas.drawBitmap(logoBitmap, null, dst, logoPaint)
            textStartX += logoW + gap
        }

        val textTopStart = rect.centerY() - (textStackH / 2f) + titleSize * 0.85f
        val y2 = textTopStart + subSize * 1.25f
        val y3 = y2 + subSize * 1.20f

        canvas.drawText(line1, textStartX, textTopStart, mainPaint)
        canvas.drawText(line2, textStartX, y2, subPaint)
        canvas.drawText(line3, textStartX, y3, subPaint)
    }

    private fun computePositionedRect(
        canvasWidth: Int,
        canvasHeight: Int,
        boxWidth: Float,
        boxHeight: Float,
        position: WatermarkPosition,
        marginPercentX: Float,
        marginPercentY: Float
    ): RectF {
        val marginX = canvasWidth * marginPercentX.coerceIn(0f, 0.25f)
        val marginY = canvasHeight * marginPercentY.coerceIn(0f, 0.25f)

        val left = when (position) {
            WatermarkPosition.TOP_LEFT, WatermarkPosition.BOTTOM_LEFT -> marginX
            WatermarkPosition.TOP_RIGHT, WatermarkPosition.BOTTOM_RIGHT -> canvasWidth - marginX - boxWidth
            WatermarkPosition.BOTTOM_BANNER -> (canvasWidth - boxWidth) / 2f
        }
        val top = when (position) {
            WatermarkPosition.TOP_LEFT, WatermarkPosition.TOP_RIGHT -> marginY
            WatermarkPosition.BOTTOM_LEFT, WatermarkPosition.BOTTOM_RIGHT, WatermarkPosition.BOTTOM_BANNER ->
                canvasHeight - marginY - boxHeight
        }
        return RectF(left, top, left + boxWidth, top + boxHeight)
    }

    private suspend fun extractPhotoMetadataAndLocation(uri: Uri): MediaExifInfo {
        var make = Build.MANUFACTURER.uppercase(Locale.ROOT)
        var model = Build.MODEL
        var focal = "24mm"
        var aperture = "f/1.8"
        var shutter = "1/125s"
        var iso = "ISO 100"
        var exifLat: Double? = null
        var exifLon: Double? = null
        var formattedDate = SimpleDateFormat("dd MMM yyyy • HH:mm", Locale("id", "ID")).format(Date())

        contentResolver.openInputStream(uri)?.use { stream ->
            val exif = ExifInterface(stream)
            exif.getAttribute(ExifInterface.TAG_MAKE)?.takeIf { it.isNotBlank() }?.let { make = it }
            exif.getAttribute(ExifInterface.TAG_MODEL)?.takeIf { it.isNotBlank() }?.let { model = it }

            exif.getAttribute(ExifInterface.TAG_FOCAL_LENGTH_IN_35MM_FILM)?.let {
                focal = "${it}mm"
            } ?: exif.getAttributeDouble(ExifInterface.TAG_FOCAL_LENGTH, 0.0).takeIf { it > 0 }?.let {
                focal = "${it.roundToInt()}mm"
            }

            exif.getAttribute(ExifInterface.TAG_F_NUMBER)?.let { aperture = "f/$it" }

            exif.getAttributeDouble(ExifInterface.TAG_EXPOSURE_TIME, 0.0).takeIf { it > 0 }?.let { exp ->
                shutter = if (exp < 1.0) "1/${(1.0 / exp).roundToInt()}s" else "${exp}s"
            }

            exif.getAttribute(ExifInterface.TAG_PHOTOGRAPHIC_SENSITIVITY)?.let { iso = "ISO $it" }

            exif.getAttribute(ExifInterface.TAG_DATETIME_ORIGINAL)?.let { raw ->
                runCatching {
                    val parsed = SimpleDateFormat("yyyy:MM:dd HH:mm:ss", Locale.US).parse(raw)
                    if (parsed != null) {
                        formattedDate = SimpleDateFormat("dd MMM yyyy • HH:mm", Locale("id", "ID")).format(parsed)
                    }
                }
            }

            exif.latLong?.let { latLon ->
                exifLat = latLon[0]
                exifLon = latLon[1]
            }
        }

        // Gunakan EXIF GPS atau fallback ke FusedLocationProviderClient pasif
        val finalCoords = geolocationManager.resolveCoordinatesWithFallback(exifLat, exifLon)
        val city = geolocationManager.reverseGeocodeOffline(finalCoords?.first, finalCoords?.second)

        return MediaExifInfo(
            deviceMake = make,
            deviceModel = model,
            focalLength = focal,
            aperture = aperture,
            shutterSpeed = shutter,
            iso = iso,
            formattedDateTime = formattedDate,
            latitude = finalCoords?.first,
            longitude = finalCoords?.second,
            resolvedCityName = city
        )
    }

    private suspend fun extractVideoMetadataAndLocation(uri: Uri): Pair<VideoSize, MediaExifInfo> {
        val retriever = MediaMetadataRetriever()
        var width = 1080
        var height = 1920
        var exifLat: Double? = null
        var exifLon: Double? = null

        try {
            retriever.setDataSource(appContext, uri)
            val rawW = retriever.extractMetadata(MediaMetadataRetriever.METADATA_KEY_VIDEO_WIDTH)?.toIntOrNull() ?: 1080
            val rawH = retriever.extractMetadata(MediaMetadataRetriever.METADATA_KEY_VIDEO_HEIGHT)?.toIntOrNull() ?: 1920
            val rotation = retriever.extractMetadata(MediaMetadataRetriever.METADATA_KEY_VIDEO_ROTATION)?.toIntOrNull() ?: 0

            if (rotation == 90 || rotation == 270) {
                width = rawH
                height = rawW
            } else {
                width = rawW
                height = rawH
            }

            val locString = retriever.extractMetadata(MediaMetadataRetriever.METADATA_KEY_LOCATION)
            if (!locString.isNullOrBlank()) {
                Regex("([+-]\\d+\\.\\d+)([+-]\\d+\\.\\d+)").find(locString)?.let { match ->
                    exifLat = match.groupValues[1].toDoubleOrNull()
                    exifLon = match.groupValues[2].toDoubleOrNull()
                }
            }
        } finally {
            retriever.release()
        }

        val finalCoords = geolocationManager.resolveCoordinatesWithFallback(exifLat, exifLon)
        val city = geolocationManager.reverseGeocodeOffline(finalCoords?.first, finalCoords?.second)
        val formattedDate = SimpleDateFormat("dd MMM yyyy • HH:mm", Locale("id", "ID")).format(Date())

        return VideoSize(width, height) to MediaExifInfo(
            deviceMake = Build.MANUFACTURER.uppercase(Locale.ROOT),
            deviceModel = Build.MODEL,
            focalLength = "24mm",
            aperture = "f/1.8",
            shutterSpeed = "60fps",
            iso = "HDR",
            formattedDateTime = formattedDate,
            latitude = finalCoords?.first,
            longitude = finalCoords?.second,
            resolvedCityName = city
        )
    }

    private fun readExifRotation(uri: Uri): Int {
        return contentResolver.openInputStream(uri)?.use { stream ->
            val exif = ExifInterface(stream)
            when (exif.getAttributeInt(ExifInterface.TAG_ORIENTATION, ExifInterface.ORIENTATION_NORMAL)) {
                ExifInterface.ORIENTATION_ROTATE_90 -> 90
                ExifInterface.ORIENTATION_ROTATE_180 -> 180
                ExifInterface.ORIENTATION_ROTATE_270 -> 270
                else -> 0
            }
        } ?: 0
    }

    private fun duplicateExifMetadata(sourceUri: Uri, targetExif: ExifInterface) {
        val sourceExif = contentResolver.openInputStream(sourceUri)?.use { ExifInterface(it) } ?: return
        for (tag in EXIF_TAGS_TO_COPY) {
            val value = sourceExif.getAttribute(tag)
            if (value != null) {
                targetExif.setAttribute(tag, value)
            }
        }
        targetExif.setAttribute(
            ExifInterface.TAG_ORIENTATION,
            ExifInterface.ORIENTATION_NORMAL.toString()
        )
        targetExif.saveAttributes()
    }

    /**
     * Penyimpanan Foto Lintas SDK:
     * - API 29+ (Android 10 - 15): Menggunakan MediaStore RELATIVE_PATH ("DCIM/AutoMark") + IS_PENDING.
     * - API 26-28 (Android 8.0 - 9.0): Menulis ke folder publik DCIM/AutoMark lalu memindai via MediaScannerConnection.
     */
    @Suppress("DEPRECATION")
    private suspend fun saveWatermarkedBitmapCrossVersion(
        bitmap: Bitmap,
        sourceUri: Uri,
        displayName: String,
        mimeType: String
    ): Uri {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
            val values = ContentValues().apply {
                put(MediaStore.MediaColumns.DISPLAY_NAME, displayName)
                put(MediaStore.MediaColumns.MIME_TYPE, mimeType)
                put(MediaStore.MediaColumns.RELATIVE_PATH, OUTPUT_SUBFOLDER)
                put(MediaStore.MediaColumns.IS_PENDING, 1)
            }
            val outputUri = contentResolver.insert(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, values)
                ?: throw IllegalStateException("Gagal membuat entri MediaStore di $OUTPUT_SUBFOLDER")

            try {
                contentResolver.openOutputStream(outputUri)?.use { out ->
                    bitmap.compress(Bitmap.CompressFormat.JPEG, 100, out)
                    out.flush()
                }
                contentResolver.openFileDescriptor(outputUri, "rw")?.use { pfd ->
                    duplicateExifMetadata(sourceUri, ExifInterface(pfd.fileDescriptor))
                }
                val publish = ContentValues().apply { put(MediaStore.MediaColumns.IS_PENDING, 0) }
                contentResolver.update(outputUri, publish, null, null)
                return outputUri
            } catch (e: Exception) {
                contentResolver.delete(outputUri, null, null)
                throw e
            }
        } else {
            val dcimDir = Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_DCIM)
            val autoMarkDir = File(dcimDir, "AutoMark").apply { if (!exists()) mkdirs() }
            val destFile = File(autoMarkDir, displayName)

            FileOutputStream(destFile).use { out ->
                bitmap.compress(Bitmap.CompressFormat.JPEG, 100, out)
                out.flush()
            }
            duplicateExifMetadata(sourceUri, ExifInterface(destFile.absolutePath))

            return scanLegacyFileToMediaStore(destFile, mimeType)
        }
    }

    @Suppress("DEPRECATION")
    private suspend fun saveWatermarkedVideoCrossVersion(
        tempFile: File,
        displayName: String,
        mimeType: String
    ): Uri {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
            val values = ContentValues().apply {
                put(MediaStore.MediaColumns.DISPLAY_NAME, displayName)
                put(MediaStore.MediaColumns.MIME_TYPE, mimeType)
                put(MediaStore.MediaColumns.RELATIVE_PATH, OUTPUT_SUBFOLDER)
                put(MediaStore.MediaColumns.IS_PENDING, 1)
            }
            val outputUri = contentResolver.insert(MediaStore.Video.Media.EXTERNAL_CONTENT_URI, values)
                ?: throw IllegalStateException("Gagal membuat entri video di $OUTPUT_SUBFOLDER")

            try {
                contentResolver.openOutputStream(outputUri)?.use { out ->
                    tempFile.inputStream().use { input -> input.copyTo(out, 64 * 1024) }
                    out.flush()
                }
                val publish = ContentValues().apply { put(MediaStore.MediaColumns.IS_PENDING, 0) }
                contentResolver.update(outputUri, publish, null, null)
                return outputUri
            } catch (e: Exception) {
                contentResolver.delete(outputUri, null, null)
                throw e
            }
        } else {
            val dcimDir = Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_DCIM)
            val autoMarkDir = File(dcimDir, "AutoMark").apply { if (!exists()) mkdirs() }
            val destFile = File(autoMarkDir, displayName)
            tempFile.copyTo(destFile, overwrite = true)
            return scanLegacyFileToMediaStore(destFile, mimeType)
        }
    }

    private suspend fun scanLegacyFileToMediaStore(file: File, mimeType: String): Uri =
        suspendCancellableCoroutine { cont ->
            MediaScannerConnection.scanFile(
                appContext,
                arrayOf(file.absolutePath),
                arrayOf(mimeType)
            ) { _, scannedUri ->
                if (cont.isActive) {
                    val safeUri = scannedUri ?: FileProvider.getUriForFile(
                        appContext,
                        "${appContext.packageName}.fileprovider",
                        file
                    )
                    cont.resume(safeUri)
                }
            }
        }

    data class VideoSize(val width: Int, val height: Int)
}
```

## 8. Penanganan Agresif Background Killer Multi-OEM (`OemBackgroundKillerHelper.kt`)

Menyediakan panduan Intent otomatis untuk mengarahkan pengguna membuka menu **"App Battery Usage / Auto-launch / Allow Background Activity"** pada **ColorOS (Oppo/Realme/OnePlus)**, **MIUI/HyperOS (Xiaomi/POCO)**, **OneUI (Samsung)**, serta **FuntouchOS/OriginOS (Vivo)**.

```kotlin
package com.automark.quickshare.service

import android.annotation.SuppressLint
import android.content.ComponentName
import android.content.Context
import android.content.Intent
import android.net.Uri
import android.os.Build
import android.os.PowerManager
import android.provider.Settings
import java.util.Locale

object OemBackgroundKillerHelper {

    fun isIgnoringBatteryOptimizations(context: Context): Boolean {
        val pm = context.getSystemService(Context.POWER_SERVICE) as PowerManager
        return pm.isIgnoringBatteryOptimizations(context.packageName)
    }

    /**
     * Meminta pengecualian Doze Mode / pengoptimalan baterai standar Android (API 26-35).
     */
    @SuppressLint("BatteryLife")
    fun requestIgnoreBatteryOptimizations(context: Context) {
        if (!isIgnoringBatteryOptimizations(context)) {
            val intent = Intent(Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS).apply {
                data = Uri.parse("package:${context.packageName}")
                addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
            }
            runCatching { context.startActivity(intent) }.onFailure {
                openAppDetailsSettings(context)
            }
        }
    }

    /**
     * Membuka halaman Auto-Launch / Background Activity Manager sesuai merk HP (Oppo, Xiaomi, Samsung, Vivo).
     */
    fun openOemAutoLaunchAndBatterySettings(context: Context) {
        val manufacturer = Build.MANUFACTURER.lowercase(Locale.ROOT)
        val candidateIntents = mutableListOf<Intent>()

        when {
            // Oppo, Realme, OnePlus (ColorOS / RealmeUI / OxygenOS)
            manufacturer.contains("oppo") || manufacturer.contains("realme") || manufacturer.contains("oneplus") -> {
                candidateIntents.addAll(
                    listOf(
                        Intent().setComponent(
                            ComponentName(
                                "com.oplus.safecenter",
                                "com.oplus.safecenter.permission.startup.StartupAppListActivity"
                            )
                        ),
                        Intent().setComponent(
                            ComponentName(
                                "com.coloros.safecenter",
                                "com.coloros.safecenter.permission.startup.StartupAppListActivity"
                            )
                        ),
                        Intent().setComponent(
                            ComponentName(
                                "com.coloros.safecenter",
                                "com.coloros.safecenter.startupapp.StartupAppListActivity"
                            )
                        ),
                        Intent().setComponent(
                            ComponentName(
                                "com.coloros.oppoguardelf",
                                "com.coloros.powermanager.fuelgaue.PowerUsageModelActivity"
                            )
                        )
                    )
                )
            }

            // Xiaomi, Redmi, POCO (MIUI / HyperOS)
            manufacturer.contains("xiaomi") || manufacturer.contains("poco") || manufacturer.contains("redmi") -> {
                candidateIntents.addAll(
                    listOf(
                        Intent("miui.intent.action.OP_AUTO_START").addCategory(Intent.CATEGORY_DEFAULT),
                        Intent().setComponent(
                            ComponentName(
                                "com.miui.securitycenter",
                                "com.miui.permcenter.autostart.AutoStartManagementActivity"
                            )
                        ),
                        Intent().setComponent(
                            ComponentName(
                                "com.miui.powerkeeper",
                                "com.miui.powerkeeper.ui.HiddenAppsConfigActivity"
                            )
                        ).apply {
                            putExtra("package_name", context.packageName)
                            putExtra("package_label", "AutoMark QuickShare")
                        }
                    )
                )
            }

            // Samsung (OneUI Device Care / Battery)
            manufacturer.contains("samsung") -> {
                candidateIntents.addAll(
                    listOf(
                        Intent().setComponent(
                            ComponentName(
                                "com.samsung.android.lool",
                                "com.samsung.android.sm.battery.ui.BatteryActivity"
                            )
                        ),
                        Intent().setComponent(
                            ComponentName(
                                "com.samsung.android.sm",
                                "com.samsung.android.sm.ui.battery.BatteryActivity"
                            )
                        )
                    )
                )
            }

            // Vivo / iQOO
            manufacturer.contains("vivo") || manufacturer.contains("iqoo") -> {
                candidateIntents.addAll(
                    listOf(
                        Intent().setComponent(
                            ComponentName(
                                "com.iqoo.secure",
                                "com.iqoo.secure.ui.phoneoptimize.AddWhiteListActivity"
                            )
                        ),
                        Intent().setComponent(
                            ComponentName(
                                "com.vivo.permissionmanager",
                                "com.vivo.permissionmanager.activity.BgStartUpManagerActivity"
                            )
                        )
                    )
                )
            }
        }

        for (intent in candidateIntents) {
            intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
            if (runCatching { context.startActivity(intent) }.isSuccess) {
                return
            }
        }

        // Fallback ke halaman App Info standar jika aktivitas spesifik OEM tidak ditemukan
        openAppDetailsSettings(context)
    }

    fun openAppDetailsSettings(context: Context) {
        val intent = Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS).apply {
            data = Uri.parse("package:${context.packageName}")
            addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
        }
        runCatching { context.startActivity(intent) }
    }
}
```

## 9. Foreground Service & Heads-Up Quick Share (`AutoMarkWatcherService.kt` & `QuickShareNotificationHelper.kt`)

Menjalankan pemantauan persisten dengan notifikasi status bar, merender foto/video baru secara berurutan, dan memunculkan **Heads-Up Notification** dengan tombol pintasan langsung ke **Instagram Stories**, **WhatsApp**, dan **TikTok**.

```kotlin
package com.automark.quickshare.service

import android.app.Notification
import android.app.NotificationChannel
import android.app.NotificationManager
import android.app.PendingIntent
import android.content.BroadcastReceiver
import android.content.Context
import android.content.Intent
import android.content.pm.ServiceInfo
import android.graphics.Bitmap
import android.graphics.BitmapFactory
import android.net.Uri
import android.os.Build
import android.util.Log
import androidx.core.app.NotificationCompat
import androidx.core.app.NotificationManagerCompat
import androidx.lifecycle.LifecycleService
import androidx.lifecycle.lifecycleScope
import com.automark.quickshare.data.location.OfflineGeolocationManager
import com.automark.quickshare.data.media.UniversalMediaContentObserver
import com.automark.quickshare.data.media.WatermarkRenderer
import com.automark.quickshare.data.preferences.WatermarkTemplateRepository
import com.automark.quickshare.domain.model.MediaCaptureEvent
import com.automark.quickshare.domain.model.MediaType
import com.automark.quickshare.domain.model.WatermarkTemplate
import com.automark.quickshare.ui.MainActivity
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.flow.collectLatest
import kotlinx.coroutines.launch
import kotlinx.coroutines.sync.Mutex
import kotlinx.coroutines.sync.withLock

class AutoMarkWatcherService : LifecycleService() {

    private lateinit var templateRepository: WatermarkTemplateRepository
    private lateinit var geolocationManager: OfflineGeolocationManager
    private lateinit var watermarkRenderer: WatermarkRenderer
    private lateinit var mediaObserver: UniversalMediaContentObserver
    private lateinit var notificationHelper: QuickShareNotificationHelper

    @Volatile
    private var currentTemplate: WatermarkTemplate = WatermarkTemplate()
    private val renderMutex = Mutex()

    companion object {
        private const val TAG = "AutoMarkWatcherService"
        private const val FOREGROUND_SERVICE_ID = 2001

        fun startService(context: Context) {
            val intent = Intent(context, AutoMarkWatcherService::class.java)
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
                context.startForegroundService(intent)
            } else {
                context.startService(intent)
            }
        }

        fun stopService(context: Context) {
            context.stopService(Intent(context, AutoMarkWatcherService::class.java))
        }
    }

    override fun onCreate() {
        super.onCreate()
        templateRepository = WatermarkTemplateRepository(this)
        geolocationManager = OfflineGeolocationManager(this)
        watermarkRenderer = WatermarkRenderer(this, geolocationManager)
        notificationHelper = QuickShareNotificationHelper(this)

        promoteToForeground()

        // Pantau perubahan konfigurasi template secara reaktif
        lifecycleScope.launch(Dispatchers.IO) {
            geolocationManager.importCustomJsonIfAvailable()
            templateRepository.templateFlow.collectLatest { updated ->
                currentTemplate = updated
            }
        }

        mediaObserver = UniversalMediaContentObserver(
            context = this,
            coroutineScope = lifecycleScope,
            debounceDelayProvider = { currentTemplate.watcherDebounceMs },
            onCameraMediaValidated = { event ->
                processDetectedMedia(event)
            }
        )
        mediaObserver.startObserving()
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        super.onStartCommand(intent, flags, startId)
        return START_STICKY
    }

    override fun onDestroy() {
        mediaObserver.stopObserving()
        geolocationManager.close()
        super.onDestroy()
    }

    private fun promoteToForeground() {
        val statusNotification = notificationHelper.buildPersistentWatcherNotification()
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.UPSIDE_DOWN_CAKE) {
            startForeground(
                FOREGROUND_SERVICE_ID,
                statusNotification,
                ServiceInfo.FOREGROUND_SERVICE_TYPE_SPECIAL_USE
            )
        } else {
            startForeground(FOREGROUND_SERVICE_ID, statusNotification)
        }
    }

    private fun processDetectedMedia(event: MediaCaptureEvent) {
        lifecycleScope.launch(Dispatchers.IO) {
            renderMutex.withLock {
                try {
                    val template = currentTemplate
                    val logoBitmap = loadLogoBitmap(template)

                    val exportedUri = when (event.mediaType) {
                        MediaType.IMAGE -> watermarkRenderer.processPhoto(
                            event = event,
                            template = template,
                            logoBitmap = logoBitmap
                        )
                        MediaType.VIDEO -> watermarkRenderer.processVideo(
                            event = event,
                            template = template,
                            logoBitmap = logoBitmap
                        )
                    }

                    logoBitmap?.recycle()

                    notificationHelper.showHeadsUpQuickShare(
                        exportedUri = exportedUri,
                        mimeType = if (event.mediaType == MediaType.IMAGE) "image/jpeg" else event.mimeType,
                        originalDisplayName = event.displayName
                    )
                } catch (e: Exception) {
                    Log.e(TAG, "Gagal merender watermark pada ${event.displayName}", e)
                }
            }
        }
    }

    private fun loadLogoBitmap(template: WatermarkTemplate): Bitmap? {
        // 1. Jika pengguna memilih logo PNG kustom dari galeri, baca dari URI tersebut
        template.customLogoUriString?.let { uriStr ->
            runCatching {
                contentResolver.openInputStream(Uri.parse(uriStr))?.use {
                    return BitmapFactory.decodeStream(it)
                }
            }
        }
        // 2. Fallback ke assets/logo_default.png
        return runCatching {
            assets.open("logo_default.png").use { BitmapFactory.decodeStream(it) }
        }.getOrNull()
    }
}

class QuickShareNotificationHelper(private val context: Context) {

    companion object {
        const val CHANNEL_PERSISTENT_WATCHER = "automark_persistent_watcher"
        const val CHANNEL_HEADSUP_SHARE = "automark_headsup_quickshare"

        const val ACTION_QUICK_SHARE = "com.automark.quickshare.ACTION_QUICK_SHARE"
        const val EXTRA_URI = "extra_uri"
        const val EXTRA_MIME = "extra_mime"
        const val EXTRA_TARGET = "extra_target"

        const val TARGET_IG_STORIES = "IG_STORIES"
        const val TARGET_WHATSAPP = "WHATSAPP"
        const val TARGET_TIKTOK = "TIKTOK"
    }

    init {
        ensureChannelsCreated()
    }

    private fun ensureChannelsCreated() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val nm = context.getSystemService(NotificationManager::class.java)

            val watcherChannel = NotificationChannel(
                CHANNEL_PERSISTENT_WATCHER,
                "AutoMark Background Watcher",
                NotificationManager.IMPORTANCE_LOW
            ).apply {
                description = "Notifikasi status bar untuk menjaga pemantau kamera tetap aktif"
                setShowBadge(false)
            }

            val headsUpChannel = NotificationChannel(
                CHANNEL_HEADSUP_SHARE,
                "AutoMark Instant Quick Share",
                NotificationManager.IMPORTANCE_HIGH
            ).apply {
                description = "Pop-up Heads-Up untuk langsung membagikan hasil watermark ke sosial media"
                enableVibration(true)
            }

            nm.createNotificationChannels(listOf(watcherChannel, headsUpChannel))
        }
    }

    fun buildPersistentWatcherNotification(): Notification {
        val openAppIntent = PendingIntent.getActivity(
            context,
            0,
            Intent(context, MainActivity::class.java),
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )

        return NotificationCompat.Builder(context, CHANNEL_PERSISTENT_WATCHER)
            .setSmallIcon(android.R.drawable.ic_menu_camera)
            .setContentTitle("AutoMark QuickShare Aktif (Offline)")
            .setContentText("Memantau foto & video baru dari kamera bawaan...")
            .setContentIntent(openAppIntent)
            .setOngoing(true)
            .setPriority(NotificationCompat.PRIORITY_LOW)
            .build()
    }

    fun showHeadsUpQuickShare(
        exportedUri: Uri,
        mimeType: String,
        originalDisplayName: String
    ) {
        val notifId = (System.currentTimeMillis() and 0xFFFFFFF).toInt()

        val igPending = createShareActionPendingIntent(notifId * 10 + 1, exportedUri, mimeType, TARGET_IG_STORIES)
        val waPending = createShareActionPendingIntent(notifId * 10 + 2, exportedUri, mimeType, TARGET_WHATSAPP)
        val ttPending = createShareActionPendingIntent(notifId * 10 + 3, exportedUri, mimeType, TARGET_TIKTOK)

        val previewBitmap = if (mimeType.startsWith("image/")) {
            runCatching {
                val opts = BitmapFactory.Options().apply { inSampleSize = 4 }
                context.contentResolver.openInputStream(exportedUri)?.use {
                    BitmapFactory.decodeStream(it, null, opts)
                }
            }.getOrNull()
        } else null

        val builder = NotificationCompat.Builder(context, CHANNEL_HEADSUP_SHARE)
            .setSmallIcon(android.R.drawable.ic_menu_share)
            .setContentTitle("Watermark Selesai: $originalDisplayName")
            .setContentText("Tersimpan di DCIM/AutoMark • Ketuk tombol untuk Quick Share:")
            .setPriority(NotificationCompat.PRIORITY_HIGH)
            .setCategory(NotificationCompat.CATEGORY_CALL)
            .setDefaults(NotificationCompat.DEFAULT_ALL)
            .setAutoCancel(true)
            .addAction(android.R.drawable.ic_menu_send, "IG Stories", igPending)
            .addAction(android.R.drawable.ic_dialog_email, "WhatsApp", waPending)
            .addAction(android.R.drawable.ic_media_play, "TikTok", ttPending)

        if (previewBitmap != null) {
            builder.setStyle(
                NotificationCompat.BigPictureStyle()
                    .bigPicture(previewBitmap)
                    .setSummaryText("Siap dibagikan ke Instagram Stories, WhatsApp, atau TikTok")
            )
        }

        try {
            NotificationManagerCompat.from(context).notify(notifId, builder.build())
        } catch (e: SecurityException) {
            Log.w("QuickShareHelper", "POST_NOTIFICATIONS belum diizinkan", e)
        }
    }

    private fun createShareActionPendingIntent(
        reqCode: Int,
        uri: Uri,
        mimeType: String,
        target: String
    ): PendingIntent {
        val intent = Intent(context, QuickShareActionReceiver::class.java).apply {
            action = ACTION_QUICK_SHARE
            putExtra(EXTRA_URI, uri.toString())
            putExtra(EXTRA_MIME, mimeType)
            putExtra(EXTRA_TARGET, target)
        }
        return PendingIntent.getBroadcast(
            context,
            reqCode,
            intent,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )
    }
}

class QuickShareActionReceiver : BroadcastReceiver() {

    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action != QuickShareNotificationHelper.ACTION_QUICK_SHARE) return

        val uriStr = intent.getStringExtra(QuickShareNotificationHelper.EXTRA_URI) ?: return
        val mime = intent.getStringExtra(QuickShareNotificationHelper.EXTRA_MIME) ?: "image/jpeg"
        val target = intent.getStringExtra(QuickShareNotificationHelper.EXTRA_TARGET) ?: return
        val mediaUri = Uri.parse(uriStr)

        val launchIntent = when (target) {
            QuickShareNotificationHelper.TARGET_IG_STORIES -> createInstagramStoriesIntent(context, mediaUri, mime)
            QuickShareNotificationHelper.TARGET_WHATSAPP -> createTargetedSendIntent(
                context, mediaUri, mime, listOf("com.whatsapp", "com.whatsapp.w4b")
            )
            QuickShareNotificationHelper.TARGET_TIKTOK -> createTargetedSendIntent(
                context, mediaUri, mime, listOf("com.ss.android.ugc.trill", "com.zhiliaoapp.musically")
            )
            else -> Intent.createChooser(
                Intent(Intent.ACTION_SEND).apply {
                    type = mime
                    putExtra(Intent.EXTRA_STREAM, mediaUri)
                    addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
                },
                "Bagikan Media AutoMark"
            )
        }

        launchIntent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_GRANT_READ_URI_PERMISSION)
        runCatching {
            context.startActivity(launchIntent)
        }.onFailure {
            val chooser = Intent.createChooser(
                Intent(Intent.ACTION_SEND).apply {
                    type = mime
                    putExtra(Intent.EXTRA_STREAM, mediaUri)
                    addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
                },
                "Bagikan Media AutoMark"
            ).apply { addFlags(Intent.FLAG_ACTIVITY_NEW_TASK) }
            context.startActivity(chooser)
        }
    }

    private fun createInstagramStoriesIntent(context: Context, uri: Uri, mimeType: String): Intent {
        val igPkg = "com.instagram.android"
        val storyIntent = Intent("com.instagram.share.ADD_TO_STORY").apply {
            setDataAndType(uri, mimeType)
            setPackage(igPkg)
            putExtra("source_application", context.packageName)
            addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
        }
        context.grantUriPermission(igPkg, uri, Intent.FLAG_GRANT_READ_URI_PERMISSION)
        return if (storyIntent.resolveActivity(context.packageManager) != null) {
            storyIntent
        } else {
            createTargetedSendIntent(context, uri, mimeType, listOf(igPkg))
        }
    }

    private fun createTargetedSendIntent(
        context: Context,
        uri: Uri,
        mimeType: String,
        candidatePackages: List<String>
    ): Intent {
        val installedPkg = candidatePackages.firstOrNull { pkg ->
            runCatching { context.packageManager.getPackageInfo(pkg, 0) }.isSuccess
        }
        return Intent(Intent.ACTION_SEND).apply {
            type = mimeType
            putExtra(Intent.EXTRA_STREAM, uri)
            if (installedPkg != null) {
                setPackage(installedPkg)
                context.grantUriPermission(installedPkg, uri, Intent.FLAG_GRANT_READ_URI_PERMISSION)
            }
            addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
        }
    }
}

class BootCompletedReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action == Intent.ACTION_BOOT_COMPLETED ||
            intent.action == "android.intent.action.QUICKBOOT_POWERON" ||
            intent.action == "com.htc.intent.action.QUICKBOOT_POWERON"
        ) {
            AutoMarkWatcherService.startService(context)
        }
    }
}
```

## 10. UI Pengaturan & Custom Template Builder (`MainActivity.kt`)

Menyediakan antarmuka lengkap berbasis Jetpack Compose (Material 3) untuk:
1. Meminta seluruh izin runtime sesuai versi Android (API 26–35) dan menyalakan `AutoMarkWatcherService`.
2. Membuka menu **App Battery Usage / Auto-Launch** otomatis untuk ColorOS, MIUI, OneUI, dan Vivo.
3. Memilih salah satu dari **4 Preset Bawaan** atau mengatur **Custom Template Builder** (posisi `TOP_LEFT`, `TOP_RIGHT`, `BOTTOM_LEFT`, `BOTTOM_RIGHT`, `BOTTOM_BANNER`, slider Opasitas Logo `0–100%`, slider Skala Ukuran Logo `% lebar foto`, slider Margin Tepi, serta pemilih file Logo PNG transparan).

```kotlin
package com.automark.quickshare.ui

import android.Manifest
import android.content.Intent
import android.os.Build
import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.compose.setContent
import androidx.activity.result.contract.ActivityResultContracts
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.verticalScroll
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.lifecycle.lifecycleScope
import com.automark.quickshare.data.preferences.WatermarkTemplateRepository
import com.automark.quickshare.domain.model.*
import com.automark.quickshare.service.AutoMarkWatcherService
import com.automark.quickshare.service.OemBackgroundKillerHelper
import kotlinx.coroutines.launch
import kotlin.math.roundToInt

class MainActivity : ComponentActivity() {

    private lateinit var repository: WatermarkTemplateRepository

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        repository = WatermarkTemplateRepository(this)

        setContent {
            MaterialTheme {
                Surface(modifier = Modifier.fillMaxSize()) {
                    AutoMarkDashboardScreen(
                        repository = repository,
                        onStartWatcher = { AutoMarkWatcherService.startService(this) },
                        onStopWatcher = { AutoMarkWatcherService.stopService(this) },
                        onOpenOemBatteryGuide = {
                            OemBackgroundKillerHelper.requestIgnoreBatteryOptimizations(this)
                            OemBackgroundKillerHelper.openOemAutoLaunchAndBatterySettings(this)
                        }
                    )
                }
            }
        }
    }
}

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun AutoMarkDashboardScreen(
    repository: WatermarkTemplateRepository,
    onStartWatcher: () -> Unit,
    onStopWatcher: () -> Unit,
    onOpenOemBatteryGuide: () -> Unit
) {
    val scope = rememberCoroutineScope()
    val templateState by repository.templateFlow.collectAsState(initial = WatermarkTemplate())
    var draft by remember(templateState) { mutableStateOf(templateState) }
    val context = androidx.compose.ui.platform.LocalContext.current

    // Launcher Izin Runtime Lintas Versi Android (API 26 - 35)
    val permissionLauncher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.RequestMultiplePermissions()
    ) {
        onStartWatcher()
    }

    // Launcher Pemilih Logo PNG Kustom dari Galeri/File Manager
    val logoPickerLauncher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.OpenDocument()
    ) { uri ->
        if (uri != null) {
            runCatching {
                context.contentResolver.takePersistableUriPermission(
                    uri,
                    Intent.FLAG_GRANT_READ_URI_PERMISSION
                )
            }
            draft = draft.copy(customLogoUriString = uri.toString())
            scope.launch { repository.saveTemplate(draft) }
        }
    }

    Column(
        modifier = Modifier
            .fillMaxSize()
            .verticalScroll(rememberScrollState())
            .padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(16.dp)
    ) {
        Text(
            text = "AutoMark QuickShare",
            style = MaterialTheme.typography.headlineSmall
        )
        Text(
            text = "Background Camera Watcher (100% Offline • API 26–35)",
            style = MaterialTheme.typography.bodyMedium
        )

        // Kartu Kontrol Watcher & Penanganan OEM Background Killer
        Card(modifier = Modifier.fillMaxWidth()) {
            Column(
                modifier = Modifier.padding(16.dp),
                verticalArrangement = Arrangement.spacedBy(10.dp)
            ) {
                Text("1. Status Service & Proteksi Baterai OEM", style = MaterialTheme.typography.titleMedium)
                Row(horizontalArrangement = Arrangement.spacedBy(12.dp)) {
                    Button(
                        onClick = {
                            val perms = buildRequiredPermissionsList()
                            permissionLauncher.launch(perms.toTypedArray())
                        }
                    ) {
                        Text("Aktifkan Watcher")
                    }
                    OutlinedButton(onClick = onStopWatcher) {
                        Text("Hentikan")
                    }
                }
                FilledTonalButton(
                    onClick = onOpenOemBatteryGuide,
                    modifier = Modifier.fillMaxWidth()
                ) {
                    Text("Buka Izin Auto-Launch & Baterai (ColorOS / MIUI / OneUI)")
                }
            }
        }

        // Kartu Pemilihan Preset Watermark
        Card(modifier = Modifier.fillMaxWidth()) {
            Column(
                modifier = Modifier.padding(16.dp),
                verticalArrangement = Arrangement.spacedBy(8.dp)
            ) {
                Text("2. Pilih Preset Watermark", style = MaterialTheme.typography.titleMedium)
                PresetStyle.entries.forEach { preset ->
                    Row(
                        verticalAlignment = Alignment.CenterVertically,
                        modifier = Modifier.fillMaxWidth()
                    ) {
                        RadioButton(
                            selected = draft.presetStyle == preset,
                            onClick = {
                                draft = draft.copy(presetStyle = preset)
                                scope.launch { repository.saveTemplate(draft) }
                            }
                        )
                        Text(text = preset.displayName)
                    }
                }
            }
        }

        // Kartu Custom Template Builder & Slider Proporsional
        Card(modifier = Modifier.fillMaxWidth()) {
            Column(
                modifier = Modifier.padding(16.dp),
                verticalArrangement = Arrangement.spacedBy(12.dp)
            ) {
                Text("3. Custom Template Builder", style = MaterialTheme.typography.titleMedium)

                OutlinedTextField(
                    value = draft.customText,
                    onValueChange = {
                        draft = draft.copy(customText = it)
                        scope.launch { repository.saveTemplate(draft) }
                    },
                    label = { Text("Teks Kustom / Nama Brand") },
                    modifier = Modifier.fillMaxWidth()
                )

                Text("Posisi Watermark: ${draft.position.name}", style = MaterialTheme.typography.labelLarge)
                Row(
                    modifier = Modifier.fillMaxWidth(),
                    horizontalArrangement = Arrangement.spacedBy(6.dp)
                ) {
                    WatermarkPosition.entries.forEach { pos ->
                        FilterChip(
                            selected = draft.position == pos,
                            onClick = {
                                draft = draft.copy(position = pos)
                                scope.launch { repository.saveTemplate(draft) }
                            },
                            label = { Text(pos.name.replace("BOTTOM_", "B_").replace("TOP_", "T_")) }
                        )
                    }
                }

                // Slider Opasitas Logo (0 - 100%)
                Text("Opasitas Logo PNG: ${draft.logoOpacityPercent}%")
                Slider(
                    value = draft.logoOpacityPercent.toFloat(),
                    onValueChange = { draft = draft.copy(logoOpacityPercent = it.roundToInt()) },
                    onValueChangeFinished = { scope.launch { repository.saveTemplate(draft) } },
                    valueRange = 0f..100f
                )

                // Slider Skala Ukuran Logo (% terhadap lebar foto)
                val logoScalePercentDisplay = (draft.logoScaleWidthPercent * 100f).roundToInt()
                Text("Skala Ukuran Logo: $logoScalePercentDisplay% dari Lebar Foto")
                Slider(
                    value = draft.logoScaleWidthPercent,
                    onValueChange = { draft = draft.copy(logoScaleWidthPercent = it) },
                    onValueChangeFinished = { scope.launch { repository.saveTemplate(draft) } },
                    valueRange = 0.03f..0.25f
                )

                // Slider Margin Tepi / Padding Relatif
                val marginPercentDisplay = (draft.marginPercentX * 100f).roundToInt()
                Text("Margin Tepi Proporsional: $marginPercentDisplay%")
                Slider(
                    value = draft.marginPercentX,
                    onValueChange = {
                        draft = draft.copy(marginPercentX = it, marginPercentY = it)
                    },
                    onValueChangeFinished = { scope.launch { repository.saveTemplate(draft) } },
                    valueRange = 0.01f..0.15f
                )

                // Slider Debounce Anti-Race Condition (500 - 1000 ms)
                Text("Debounce Penulisan Kamera: ${draft.watcherDebounceMs} ms")
                Slider(
                    value = draft.watcherDebounceMs.toFloat(),
                    onValueChange = { draft = draft.copy(watcherDebounceMs = it.toLong()) },
                    onValueChangeFinished = { scope.launch { repository.saveTemplate(draft) } },
                    valueRange = 500f..1000f
                )

                Button(
                    onClick = { logoPickerLauncher.launch(arrayOf("image/png")) },
                    modifier = Modifier.fillMaxWidth()
                ) {
                    Text("Pilih File Logo PNG Transparan")
                }
            }
        }
    }
}

private fun buildRequiredPermissionsList(): List<String> {
    val list = mutableListOf(
        Manifest.permission.ACCESS_COARSE_LOCATION,
        Manifest.permission.ACCESS_FINE_LOCATION
    )
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
        list.add(Manifest.permission.READ_MEDIA_IMAGES)
        list.add(Manifest.permission.READ_MEDIA_VIDEO)
        list.add(Manifest.permission.POST_NOTIFICATIONS)
    } else {
        list.add(Manifest.permission.READ_EXTERNAL_STORAGE)
        if (Build.VERSION.SDK_INT <= Build.VERSION_CODES.P) {
            list.add(Manifest.permission.WRITE_EXTERNAL_STORAGE)
        }
    }
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
        list.add(Manifest.permission.ACCESS_MEDIA_LOCATION)
    }
    return list
}
```