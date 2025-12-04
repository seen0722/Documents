# Part A · Chapter 4 --- VINTF（Vendor Interface）與 HAL 版本管理

## 4.1 VINTF 是什麼？

VINTF（Vendor Interface）是 Android 8（Oreo）引入的系統/vendor
分離機制，用來保證： - system image 可以獨立更新 - vendor image
不需同步更新 - system 與 vendor 之間的 HAL API 版本完全相容 - HAL
介面穩定（version/hash）

它是 Android Treble 的核心機制，確保 framework ↔ vendor HAL 之間永不
break。

------------------------------------------------------------------------

## 4.2 VINTF 的兩大核心檔案

### 4.2.1 Framework Manifest

路徑：

    /system/etc/vintf/manifest.xml
    /system_ext/etc/vintf/manifest.xml

Framework 宣告： - 需要哪些 HAL - 版本是多少 - instance 名稱

範例：

``` xml
<hal format="aidl">
    <name>android.hardware.power</name>
    <version>3</version>
    <interface>
        <name>IPower</name>
        <instance>default</instance>
    </interface>
</hal>
```

------------------------------------------------------------------------

### 4.2.2 Vendor Manifest

路徑：

    /vendor/etc/vintf/manifest.xml

Vendor 宣告： - 提供哪些 HAL - HAL 實作的版本 - instance 名稱 -
transport（aidl/hidl）

------------------------------------------------------------------------

## 4.3 VINTF 如何檢查 HAL 相容性？

### 4.3.1 Manifest Matching

Framework_manifest 需要的 HAL\
必須能在 vendor_manifest 內找到相同版本。

若 framework 想用 v3，但 vendor 只有 v2 → FAIL

------------------------------------------------------------------------

### 4.3.2 HAL API Stability

AIDL： - InterfaceVersion - InterfaceHash

HIDL： - @version（1.0 / 1.1 / 2.0）

------------------------------------------------------------------------

### 4.3.3 FCM（Framework Compatibility Matrix）

路徑：

    /system/etc/vintf/compatibility_matrix.xml

描述 framework 對 HAL 的最低需求版本。

例如：

``` xml
<hal format="aidl">
    <name>android.hardware.power</name>
    <version>3</version> <!-- minimum -->
</hal>
```

------------------------------------------------------------------------

## 4.4 System / Vendor 升級與 VINTF 的關係

### Scenario 1：System OTA 升級

-   system 升級
-   vendor 不變
-   HAL 介面必須保持向後相容（stable）

### Scenario 2：Vendor 升級 HAL

-   vendor 升級 HAL
-   system 不變
-   不允許移除 API（HIDL = new @version，AIDL = 增加 @version）

------------------------------------------------------------------------

## 4.5 VINTF 檔案組成（完整列表）

system：

    /system/etc/vintf/manifest.xml
    /system/etc/vintf/compatibility_matrix.xml

system_ext：

    /system_ext/etc/vintf/manifest.xml

vendor：

    /vendor/etc/vintf/manifest.xml
    /vendor/etc/vintf/compatibility_matrix.xml

------------------------------------------------------------------------

## 4.6 如何檢查 VINTF 是否一致？

### 4.6.1 lshal（最常用）

    adb shell lshal

顯示： - vendor manifest - framework manifest - vendor matrix -
framework matrix

------------------------------------------------------------------------

### 4.6.2 dumpsys VINTF

    adb shell dumpsys android.os.VintfService

------------------------------------------------------------------------

### 4.6.3 vintf_tool

部分裝置支援：

    adb shell vintf

------------------------------------------------------------------------

## 4.7 HAL 版本 mismatch 的典型錯誤

### ❌ 1. framework \> vendor 版本

framework 需要 power v3\
vendor 只有 power v2\
➡️ FAIL

------------------------------------------------------------------------

### ❌ 2. vendor 跑了多個版本 HAL instance

如：

    usb@1.1::IUsb/default
    usb@1.2::IUsb/default

➡️ framework 混淆 → FAIL

------------------------------------------------------------------------

### ❌ 3. AIDL InterfaceHash mismatch

API 签名變動 → FAIL

------------------------------------------------------------------------

### ❌ 4. HAL 名稱拼錯

如：

    android.hardware.light
    android.hardware.lights

➡️ 不匹配 → FAIL

------------------------------------------------------------------------

### ❌ 5. service 未註冊成功

framework 宣告：

    IPower/default

service list 找不到：

    android.hardware.power.IPower/default

➡️ HAL service 未啟動

------------------------------------------------------------------------

## 4.8 VINTF Debug 流程（工程師實務）

### Step 1：查 HAL service

AIDL：

    service list | grep hardware

HIDL：

    lshal | grep hardware

### Step 2：查 HAL 版本

AIDL：

    dumpsys android.hardware.<hal>.I<Interface>/default

HIDL：

    lshal | grep <hal>

### Step 3：比對 manifest

    cat /vendor/etc/vintf/manifest.xml
    cat /system/etc/vintf/manifest.xml

### Step 4：查 VINTF mismatch

    dumpsys android.os.VintfService | grep -i fail

### Step 5：查 HAL 啟動失敗原因

    logcat | grep -i hal
    dmesg | grep -i binder

------------------------------------------------------------------------

## Chapter 4 小結

-   VINTF 是 system/vendor HAL 的契約\
-   framework & vendor manifest 必須一致\
-   AIDL 用 InterfaceVersion/Hash\
-   HIDL 用 @version\
-   VINTF mismatch 會導致 boot fail / CTS fail\
-   lshal / dumpsys VINTF 是最重要工具
