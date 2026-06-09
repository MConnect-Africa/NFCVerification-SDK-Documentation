# NfcDocVerification Library

A comprehensive In-House Android library for scanning Machine Readable Zones (MRZ) from identity documents and reading data from NFC-enabled biometric IDs.

## Features
- **MRZ Scanning**: Uses Google ML Kit for fast and accurate text recognition of MRZ data from document images.
- **NFC Reading**: Implements ICAO Doc 9303 standards for secure communication with biometric chips via NFC.
- **Progress Tracking**: Provides real-time updates on the NFC reading process (Authenticating, Connecting, Reading Data, etc.).
- **Data Parsing**: Automatically parses MRZ and NFC data into a structured `DocumentData` object, including personal details and the face photo.

## Installation

### 1. Add GitHub Packages Repository

Add the GitHub Packages repository to your project's `settings.gradle.kts` or `build.gradle` file:

```kotlin
// In settings.gradle.kts
dependencyResolutionManagement {
   repositories {
      // Other repositories...
      maven {
         name = "GitHubPackages"
         url = uri("maven_package_url")
         credentials {
            username = project.findProperty("gpr.user") as String? ?: System.getenv("GITHUB_USERNAME")
            password = project.findProperty("gpr.key") as String? ?: System.getenv("GITHUB_TOKEN")
         }
      }
   }
}
```
### 2. Add Authentication Credentials

Add your GitHub username and personal access token to your `local.properties` file:

```properties
gpr.user=YOUR_GITHUB_USERNAME
gpr.key=YOUR_GITHUB_PERSONAL_ACCESS_TOKEN
```

### 3. Add the Dependency

Add the dependency to your app's `build.gradle.kts` file:

```kotlin
dependencies {
   implementation("co.isocel:nfcdocverification:<version>")
}
```

## Setup

### 1. Permissions
Add the following permissions to your `AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.NFC" />
<uses-permission android:name="android.permission.CAMERA" />

<uses-feature android:name="android.hardware.nfc" android:required="true" />
<uses-feature android:name="android.hardware.camera" android:required="false" />
```

### 2. Initialize the Library
Initialize the `NfcDocVerification` object in your Activity's `onCreate`:

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    NfcDocVerification.initialize(this)
}
```

## Usage

### 1. Scanning the MRZ
To read an NFC chip, you first need the MRZ data (Document Number, Date of Birth, and Date of Expiry) to derive the BAC (Basic Access Control) keys. You do not need to use this
for library functionality but could be you want to display it to the user.

Using CameraX, you can set up the `mrzImageAnalysis`:

```kotlin
val cameraExecutor = Executors.newSingleThreadExecutor()
val mrzImageAnalysis = NfcDocVerification.mrzImageAnalysis(
    cameraExecutor = cameraExecutor,
    onMrzDetected = { bacKeys ->
        // MRZ detected! You can now proceed to NFC reading.
        // bacKeys contains documentNumber, dateOfBirth, and dateOfExpiry
    },
    onPartialDetection = { mrzText ->
        // You can alert the users to hold still 
    }
) 

// Bind mrzImageAnalyzer to your CameraX lifecycle
cameraProvider.bindToLifecycle(
    lifecycleOwner,
    CameraSelector.DEFAULT_BACK_CAMERA,
    preview,
    mrzImageAnalysis
)
```

### 2. Reading the NFC Chip

#### Step A: Enable NFC Foreground Dispatch
When you're ready to read the chip, enable foreground dispatch to capture NFC intents:

```kotlin
NfcDocVerification.initNfc()
// Or specifically
NfcDocVerification.initNfc(
    shouldEnableForegroundDispatch = false
)
// Then enable it manually
NfcDocVerification.enableForegroundDispatch()
```

Don't forget to disable it when not needed (e.g., in `onPause` or `onDispose`):
```kotlin
NfcDocVerification.disableForegroundDispatch()
```

#### Step B: Handle the NFC Intent
In your `onNewIntent`, extract the `intent` and call `readChip`:

```kotlin
override fun onNewIntent(intent: Intent) {
    super.onNewIntent(intent)
    lifecycleScope.launch {
        NfcDocVerification.readChip(intent)
    }
}
```

#### Step C: Observe Progress
You can collect the `nfcProgress` StateFlow to update your UI:

```kotlin
lifecycleScope.launch {
    NfcDocVerification.nfcProgress.collect { progress ->
        when (progress) {
            is NfcReadProgress.Connecting -> // Show "Connecting..."
            is NfcReadProgress.ReadingPersonalData -> // Show "Reading Data..."
            is NfcReadProgress.ReadingPhoto -> // Show "Reading Photo..."
            is NfcReadProgress.Done -> {
                val data = progress.documentData
                // Successfully read: data.fullName, data.facePhoto, etc.
            }
            is NfcReadProgress.Error -> // Show error message
            null -> // Idle
        }
    }
}
```

## Data Model: `DocumentData`
The `DocumentData` object contains the following information (if available on the chip):
- `fullName`
- `surname`
- `givenNames`
- `documentNumber`
- `issuingState`
- `nationality`
- `gender`
- `dateOfBirth` (and `formattedDob`)
- `dateOfExpiry` (and `formattedExpiry`)
- `facePhoto` (as a `Bitmap`)
- `mrzRaw`
- `county`
- `subCounty`
- `division`
- `location`
- `subLocation`

## Requirements
- Android API 26 (Android 8.0) or higher.
- A device with NFC support.
- Camera access for MRZ scanning.
