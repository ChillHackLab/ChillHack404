# Security Analysis Report  
## WebView‑Based Android Application (Web2Native Wrapper)

**Analysis Type:** Static Analysis & Network Reconnaissance  
**Platform:** Android  
**Risk Level:** High (Capability‑Based Assessment)

---

## Disclaimer

This repository is provided for **educational and defensive security research purposes only**.

All findings are based on static reverse engineering and observation of publicly accessible
resources at the time of analysis.  
No live exploitation, payload execution, or malicious infrastructure is operated or controlled
by the author.

This report describes **technical capabilities and potential security risks only** and does not
assert intent, attribution, or ownership of the analyzed application.

---

## 1. Executive Summary

The analyzed Android application presents itself as a third-party app distribution tool.  
Static analysis shows that the application is not a fully native implementation,
but operates as a **WebView‑based wrapper** built using the **Web2Native framework**.

The application loads remote web content and exposes native Android functionality
to JavaScript through a `JavascriptInterface` bridge.  
This design enables the loaded web content to invoke **high‑risk system capabilities**,
including file downloads, triggering installation intents, accessing device identifiers,
and interacting with SMS‑related APIs.

While a previously embedded backend server is no longer active, the application
continues to rely on a live web interface for behavior control, which represents a
persistent security risk as long as the remote content remains accessible.

---

## 2. Sample Identification

| Field | Value |
| --- | --- |
| Package Name | `com.web2native.*` |
| File Type | Android APK |
| SHA‑256 | `6d6239544d4493f235836bda8fc38e4e1e1dba3bff7d0aaad5176c68bb34aae7` |
| Architecture | WebView + Native Bridge |

---

## 3. Static Analysis

### 3.1 Manifest & Permissions Review

The application requests permissions and configuration flags that exceed the requirements
of a basic WebView wrapper.

Notable findings include:

- **`android.permission.WRITE_EXTERNAL_STORAGE`**  
  Enables writing downloaded content to external storage.

- **`android:usesCleartextTraffic="true"`**  
  Explicitly allows unencrypted HTTP traffic, bypassing modern Android defaults
  and increasing exposure to network-based manipulation.

- **Broad Android Version Support (API 23+)**  
  Allows execution on older Android versions that may lack recent security patches.

---

### 3.2 Embedded Configuration & Identifiers

The following hardcoded identifiers were identified during static analysis:

- Firebase Realtime Database  
  `https://alpha-af0d2.firebaseio.com` (currently deactivated)

- Firebase Storage Bucket  
  `alpha-af0d2.appspot.com`

- Third‑party SDK identifiers (e.g., Facebook App ID, Google API keys)

These artifacts indicate the application was designed to integrate with
external cloud services and third-party platforms.

---

## 4. Code Analysis (Reverse Engineering)

The application codebase is obfuscated using ProGuard/R8, resulting in short
class names such as `bb`, `ca`, and `ad`.  
Despite this, core functionality was identifiable through control‑flow analysis.

---

### 4.1 Web‑to‑Native Bridge (`bb/n1.java`)

The most security‑relevant component is the class responsible for exposing
native functionality to JavaScript via the `@JavascriptInterface` annotation.

- **Injected Interface Name:** `WebToNativeInterface`
- **Injection Point:** `WebView.addJavascriptInterface()`

This bridge allows JavaScript code loaded from remote web content to directly
invoke native Android methods.

#### Exposed Capabilities (Selected)

- **File Download & Handling**
  - Methods capable of downloading arbitrary data and writing it to storage.

- **Installation Triggers**
  - Use of `FileProvider` and installation intents to prompt APK installation.

- **Device Information Access**
  - Retrieval of installation identifiers and environment metadata.

- **SMS‑Related APIs**
  - Presence of methods that interact with SMS registration and configuration.
  - No complete end‑to-end interception flow was confirmed through static analysis alone.

---

### 4.2 Download & File Handling Pipeline

The application implements a custom data transfer flow:

- **Downloader (`bb/e1.java`)**
  - Uses `HttpURLConnection` to retrieve remote content as raw byte streams.
  - No cryptographic verification or integrity checks were observed.

- **File Writer (`bb/f1.java`)**
  - Writes downloaded byte arrays directly to cache or external storage locations.

This design enables remote content to be delivered and prepared for installation
without relying on standard browser download safeguards.

---

### 4.3 Application Entry Point

The `MainActivity` initializes the WebView and immediately loads a remote URL
upon application launch:

