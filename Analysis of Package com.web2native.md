# Static Analysis Report: `com.web2native` / `1d5b2b...`

## Disclaimer

> **This repository is for educational and defensive security research purposes only.**
> All analysis is based on static reverse engineering of publicly available samples.
> No live exploitation, payload execution, or malicious infrastructure is operated
> or controlled by the author.
> The conclusions represent a technical risk assessment based on code capabilities, not an accusation of intent.

---

## 1. Sample Information

| Attribute | Detail |
| --- | --- |
| **SHA-256** | `1d5b2b87512ce21a691b9638b482c553594fe3dc7eb908a103e63c0c2c044d65` |
| **Package Name** | `com.web2native` (Derived from code structure) |
| **File Type** | Android Package (APK) |
| **Architecture** | Web2Native Wrapper |

---

## 2. Initial Assessment & File Identification

### 2.1 File Verification

Initial attempts to unzip the sample failed due to potential obfuscation or file structure anomalies. We verified the file signature to confirm it was a valid archive.

**Command:**

```bash
file 6d6239544d4493f235836bda8fc38e4e1e1dba3bff7d0aaad5176c68bb34aae7.apk

```

**Finding:** The file is identified as a Zip archive data, consistent with the APK format.

### 2.2 Archive Extraction

We successfully extracted the contents to inspect the internal structure.

**Command:**

```bash
unzip 1d5b2b...d65.apk -d unzip_output

```

**Output Artifacts:**

* `classes.dex` (Dalvik Bytecode)
* `resources.arsc` (Compiled Resources)
* `AndroidManifest.xml` (Binary XML)

---

## 3. Network & Infrastructure Analysis

### 3.1 Hardcoded Remote Infrastructure

We analyzed binary resources to identify backend infrastructure associated with the application.

**Command:**

```bash
strings resources.arsc | grep -C 5 "alpha-af0d2"

```

**Output Findings:**

```text
alpha-af0d2
alpha-af0d2.appspot.com
"https://alpha-af0d2.firebaseio.com"
fb722156285706830

```

* **Observation:** The application references a **Firebase Realtime Database** (`alpha-af0d2`) and a Google Cloud Storage bucket.

### 3.2 Infrastructure Status Check

We probed the identified Firebase endpoint to assess its activity status.

**Command:**

```bash
curl "https://alpha-af0d2.firebaseio.com/.json?print=pretty"

```

**Output:**

```json
{
  "error" : "The Firebase database 'alpha-af0d2' has been deactivated."
}

```

* **Technical Assessment:** The remote control infrastructure has been deactivated by the service provider.

---

## 4. Entry Point Analysis (Web Context)

### 4.1 Identifying the Primary URL

Using `jadx` to decompile the `classes.dex` and `grep` to search the source, we identified the initial URL loaded by the `MainActivity`.

**Command:**

```bash
grep -r "sfield_a1 =" source_code/sources/com/web2native/MainActivity.java

```

**Finding:**
The application contains logic to load a hardcoded string `K0`:

```java
public static String K0 = "https://tutuapp.store/androidstore/";

```

* **Observation:** The domain `tutuapp.store` appears to mimic a known third-party application store ("TutuApp"), but is hosted on a different TLD (`.store` vs `.vip` or `.com`).

### 4.2 Web Content Inspection

We analyzed the source code of the target URL to understand how it interacts with the application wrapper.

**Command:**

```bash
curl -L "https://tutuapp.store/androidstore/"

```

**Findings:**

1. **Redirection/Hosting:** The site pulls assets (JS/CSS) from `tutuapp-vip.com`.
2. **Platform:** The site runs on WordPress.
3. **Dynamic Loading:** The site utilizes `lazyload.min.js` and iframe injection techniques, which can be used to dynamically load content or payloads after the initial page load.

---

## 5. Code Logic & Bridge Analysis (High Risk Capabilities)

The application functions as a high-privilege shell. It exposes native Android capabilities to the web context via a **Javascript Interface**.

### 5.1 The Web-to-Native Bridge

We identified the class `bb.n1` as the bridge interface. It is registered to the WebView under the name `"WebToNativeInterface"`.

**Command:**

```bash
grep -r "addJavascriptInterface" source_code/sources/com/web2native/MainActivity.java

```

**Output:**

```java
webView.addJavascriptInterface(this.f5245p0, "WebToNativeInterface");

```

### 5.2 Exposed Native Methods

We analyzed `bb/n1.java` to enumerate the methods exposed to the web. The presence of the `@JavascriptInterface` annotation allows these methods to be called directly by any JavaScript running on the loaded page.

**Command:**

```bash
grep -B 1 "@JavascriptInterface" source_code/sources/bb/n1.java

```

**Output:** The scan revealed numerous exposed methods. Further manual inspection confirmed the following capabilities:

#### A. File Download & Execution (Dropper Capability)

The application can download files to the public storage and trigger the installation intent.

**Code Artifacts (from `bb/n1.java` & `bb/e1.java`):**

* **`downloadFile(String url)`**: Triggers a standard HTTP download.
* **`installApp(String path)`**: Utilizes `FileProvider` to trigger `Intent.ACTION_VIEW` with the MIME type `application/vnd.android.package-archive`.
* **`getBase64FromBlobData`**: Suggests the ability to reconstruct binary payloads delivered via web blobs.

#### B. Telephony Data Exfiltration (Spyware Capability)

The interface exposes methods to access sensitive device identifiers.

**Command:**

```bash
grep -Ei "device|phone" source_code/sources/bb/n1.java

```

**Findings:**

* **`getDeviceInfo()`**: Retrieves installation details and device metadata.
* **`TelephonyManager` access**: Logic exists to access the device's telephony services, typically used to harvest the **IMEI**, **IMSI**, or **Phone Number**.

#### C. SMS Interaction

The interface contains specific methods for handling Short Message Service (SMS) functions.

**Command:**

```bash
grep -Ei "sms" source_code/sources/bb/n1.java

```

**Findings:**

* `registerForSMS()`
* `setSMSNumber(String str)`
* *Risk:* These methods allow the remote web operator to potentially intercept OTPs or send messages without direct user interaction beyond the web interface.

---

## 6. Technical Conclusion & Risk Assessment

Based on the static analysis artifacts, this application is classified as **High Risk**.

### Key Risk Factors:

1. **Unrestricted Bridge:** The application exposes a "Javascript Bridge" (`WebToNativeInterface`) that grants the loaded website (`tutuapp.store`) direct access to native OS functions.
2. **Dropper Capabilities:** The code includes specific logic to download, save, and prompt the installation of APK files (`application/vnd.android.package-archive`). This allows the remote server to deploy secondary payloads.
3. **Data Harvesting:** The exposed interface allows for the exfiltration of persistent device identifiers and telephony data.
4. **Obfuscation:** The application uses ProGuard/R8 obfuscation (e.g., class names `bb/n1`, `bb/f1`) and mimics legitimate libraries to hinder analysis.

**Technical Summary:**
The application serves as a conduit. While the APK itself may not contain the final malicious payload, it provides the necessary privileges and mechanisms for a remote web operator to execute arbitrary code (via APK installation) and exfiltrate data from the device. The primary control node (Firebase) is offline, but the web entry point remains active.
