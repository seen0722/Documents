# Part A · Chapter 3 --- HIDL HAL 架構與運作原理

## 3.1 HIDL 是什麼？

HIDL（HAL Interface Definition Language）是 Android 8（Oreo）推出的 HAL
技術，為 Project Treble 而設計。其目的在於： - 分離 system/vendor - 使用
IPC（HwBinder） - 使 HAL 接口可版本化（@1.x、@2.x）

HIDL 的服務名稱格式：

    android.hardware.power@1.3::IPower/default
    android.hardware.camera.provider@2.6::ICameraProvider/legacy/0

**只要 service 名稱有 @X.Y，就代表是 HIDL HAL。**

------------------------------------------------------------------------

## 3.2 HIDL HAL 的 IPC：HwBinder

  HAL 類型   IPC 技術       IPC Manager
  ---------- -------------- ----------------------
  AIDL       Binder         ServiceManager
  **HIDL**   **HwBinder**   **hwservicemanager**

HwBinder 與 Binder 相容，但為 HAL 特別改良的版本。

------------------------------------------------------------------------

## 3.3 HIDL 架構（IPC 全流程）

    App
      ↓
    Framework Service (Java)
      ↓ JNI
    Native Framework (C++)
      ↓
    HIDL Client  (BpInterface)
      ↓ HwBinder IPC
    HIDL Service (BnInterface)
      ↓
    Vendor HAL implementation
      ↓
    Kernel driver

------------------------------------------------------------------------

## 3.4 HIDL Interface 與版本 (@1.0 / @1.1 / @2.0)

HIDL 介面定義於：

    hardware/interfaces/<module>/<version>/

例子：

    hardware/interfaces/power/1.3/IPower.hal
    hardware/interfaces/camera/provider/2.6/ICameraProvider.hal

版本規則： - 主版本增長 → 不相容修改（1.x → 2.x） - 次版本增長 → API
新增（2.5 → 2.6 → 2.7）

------------------------------------------------------------------------

## 3.5 HIDL 介面語法

範例：

``` hal
package android.hardware.power@1.3;

interface IPower {
    setInteractive(bool interactive);
    powerHint(PowerHint hint);
};
```

經 `hidl-gen` 產生： - Bp（client 端） - Bn（server 端） - 介面 C++
header

------------------------------------------------------------------------

## 3.6 HIDL 的兩種模式：Binderized / Passthrough

### 3.6.1 Binderized（主流）

    Client ↔ HwBinder ↔ HAL Service ↔ Driver

Service 名稱出現在：

    android.hardware.power@1.3::IPower/default

### 3.6.2 Passthrough（舊 HAL）

    Client ↔ HAL shared library ↔ Driver

在 `lshal` 中會出現：

    PASSTHROUGH HALS:
    android.hardware.foo@1.0::IFoo/default (passthrough)

已逐步淘汰。

------------------------------------------------------------------------

## 3.7 HIDL Service 啟動流程

### ① init.rc 啟動 HAL

    /vendor/etc/init/android.hardware.power@1.3-service.rc

``` rc
service vendor.power-hal-1-3 /vendor/bin/hw/android.hardware.power@1.3-service
    class hal
    user root
    group root
```

------------------------------------------------------------------------

### ② HAL 在 main() 中註冊到 hwservicemanager

``` cpp
configureRpcThreadpool(1, true);
sp<IPower> service = new Power();
service->registerAsService();  // 重要
joinRpcThreadpool();
```

------------------------------------------------------------------------

### ③ Framework 查詢 HAL

``` cpp
sp<IPower> hal = IPower::getService();
```

------------------------------------------------------------------------

## 3.8 如何確認 HIDL HAL 是否正在運行

### ✔ 方法 1：lshal（最重要）

    adb shell lshal | grep power

輸出：

    android.hardware.power@1.3::IPower/default

### ✔ 方法 2：hwservicemanager

    adb shell dumpsys hwservicemanager | grep -A 5 power

### ✔ 方法 3：查看 HAL process

    adb shell ps -A | grep power

------------------------------------------------------------------------

## 3.9 HIDL Debug 技巧

### ✔ 查看所有 HAL

    adb shell lshal

### ✔ 查看某個 HAL 詳細資訊

    lshal -i android.hardware.camera.provider@2.6::ICameraProvider/legacy/0

### ✔ 查看 VINTF 與 HAL 版本對應

    adb shell cat /vendor/etc/vintf/manifest.xml
    adb shell cat /system/etc/vintf/manifest.xml

### ✔ 檢查 binder 問題

    logcat | grep -i HwBinder
    dmesg | grep binder

------------------------------------------------------------------------

## 3.10 Qualcomm 平台的 HIDL HAL

常見路徑：

    /vendor/bin/hw/
    /vendor/lib64/hw/
    /vendor/etc/init/
    /vendor/etc/vintf/

Qualcomm 常見 HIDL HAL：

  模組              版本
  ----------------- -----------------------
  Camera Provider   2.4 / 2.5 / 2.6 / 2.7
  Audio HAL         5.x / 6.x
  Health HAL        2.0 / 2.1
  Power HAL（舊）   1.2 / 1.3

AIDL 正逐步取代，但 Camera/Audio 等仍是 HIDL 主戰場。

------------------------------------------------------------------------

## Chapter 3 小結

-   HIDL 使用 HwBinder（非一般 Binder）\
-   Service 名稱含 @x.y → HIDL\
-   有 Binderized / Passthrough 兩種模式\
-   HAL 使用 registerAsService() 註冊\
-   使用 lshal 最準確查看 HIDL HAL\
-   Qualcomm 大部分舊 HAL 仍使用 HIDL
