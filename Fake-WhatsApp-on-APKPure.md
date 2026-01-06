# Malware Deep-Dive Analysis Report  
## Android Trojan Impersonating WhatsApp (Ad-Fraud)

---

## 1. Executive Summary

This report documents the technical analysis of an Android APK distributed via **APKPure** that presents itself as **“WhatsApp”** using the official WhatsApp name and logo, while being signed by an unrelated developer and using a non-official package name.

Based on static analysis, smali inspection, and native library reverse engineering, the application:

- Does **not** implement any legitimate WhatsApp messaging functionality
- Is built on a **game engine (Buildbox / cocos2d)**
- Monetizes user interaction through advertisement SDK invocation
- Exhibits behavior consistent with **brand impersonation** and **ad-fraud**

Although the latest observed update dates back to **2017**, the sample remains relevant because it is still distributed and installable. Age does not mitigate risk.

---

## 2. Sample Information

| Field | Value |
|------|------|
| Distribution platform | APKPure |
| Display name | WhatsApp |
| Display icon | Official WhatsApp logo |
| Package name | `com.ibrawhat.dayz` |
| Official WhatsApp package | `com.whatsapp` |
| Developer | Unrelated third party |
| Last update | 2017 |

---

## 3. Brand Impersonation Analysis

### 3.1 Package Name Verification

**Command executed**
```bash
cat AndroidManifest.xml | grep package
````

**Raw output**

```xml
package="com.ibrawhat.dayz"
```

**Analysis**

* Official WhatsApp applications are distributed exclusively under `com.whatsapp`
* This APK uses a completely unrelated package name
* Indicates deliberate decoupling of user-facing identity and system-level identifiers

---

### 3.2 Internal Application Name

**Command executed**

```bash
grep -r "app_name" res/values/strings.xml
```

**Raw output**

```xml
<string name="app_name">New App</string>
```

**Analysis**

* Internal name does not reference WhatsApp
* Branding is applied only at the presentation layer
* Confirms intentional UI-level impersonation

---

## 4. Functional Capability Mismatch

### 4.1 Messaging & Communication Components

**Command executed**

```bash
grep -R "xmpp\|signal\|chat\|encrypt" .
```

**Raw output**

```text
(no results)
```

**Analysis**

* No messaging, encryption, or signaling components found
* Not a modified or third-party WhatsApp client

---

### 4.2 Game Engine Logic Presence

**Command executed**

```bash
grep -R "player\|level\|score" smali | wc -l
```

**Raw output**

```text
6242
```

**Analysis**

* Extensive game-related logic present
* Strong mismatch with a messaging application
* Confirms repackaged game template

---

## 5. Advertisement Monetization & Ad-Fraud Indicators

### 5.1 Advertisement Trigger Logic

**Command executed**

```bash
radare2 libplayer.so
pdf @ PTPObjectAssetPowerup::activatePowerup
```

**Disassembly excerpt**

```asm
call sym.imp.sprintf
call dword [ecx + 0x198] ; Ad SDK bridge
```

**Analysis**

* User interactions are directly mapped to advertisement SDK calls
* Behavior aligns with ad-fraud monetization patterns

---

## 6. Native Library Analysis

### 6.1 Runtime Environment Awareness

**Command executed**

```bash
pdf @ sym.CocosDenshion::android::AndroidJavaEngine::AndroidJavaEngine
```

**Disassembly excerpt**

```asm
call sym.imp.__system_property_get
call sym.imp.atoi
cmp edi, 0x15
```

**Analysis**

* Reads system properties to determine runtime environment
* Indicates environment awareness at the native layer

---

## 7. Obfuscation & Resource Protection

### 7.1 Dynamic Encryption Key Usage

**Command executed**

```bash
nm -D libplayer.so | grep EncryptionKey
```

**Raw output**

```text
_ZN7cocos2d8ZipUtils21ccSetPvrEncryptionKeyEjjjj
_ZN7cocos2d8ZipUtils16s_uEncryptionKeyE
```

**Analysis**

* Resources are decrypted dynamically at runtime
* Encryption keys are not stored in plaintext
* Increases resistance to static analysis

---

### 7.2 Base64 Resource Decoding Path

**Command executed**

```bash
axt sym.base64Decode
```

**Raw output**

```text
method.cocos2d::CCTMXMapInfo.endElement
```

**Analysis**

* Base64 decoding used during resource loading
* No direct evidence of executable payload decoding observed

---

## 8. Update Age & Risk Consideration

* Last update observed in **2017**
* Application remains downloadable and installable
* Monetization logic does not depend on ongoing updates
* Age does not reduce associated risk

---

## 9. Final Classification & Conclusion

### Malware Classification

**Android Trojan — Brand Impersonation + Ad-Fraud**

### Conclusion

The analyzed APK deliberately impersonates WhatsApp through its name and logo while implementing no legitimate messaging functionality. The application is fundamentally a repackaged game engine designed to monetize user interactions via advertisement SDKs.

The combination of brand impersonation, functional mismatch, and ad-centric behavior supports classification as an Android trojan focused on deceptive distribution and ad-fraud revenue generation.

---

*This report is provided for security research and defensive analysis purposes only.*
