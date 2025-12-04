# Part A · Chapter 2 --- AIDL HAL 架構與運作原理

## 2.1 AIDL HAL 是什麼？

AIDL HAL 是 Android 11 後 Google 推出的新一代 HAL 技術，使用與 Framework
相同的 Binder IPC，並具備穩定介面（Stable
Interface）、版本控管、可演進性，是目前 Google 主推的 HAL 架構。

------------------------------------------------------------------------

## 2.2 AIDL HAL 的核心組件

1.  AIDL Interface (`I<Module>.aidl`)
2.  HAL Service（C++ 實作）
3.  VINTF manifest（系統宣告 HAL）

------------------------------------------------------------------------

## 2.3 AIDL HAL 的目錄結構

    hardware/interfaces/<module>/aidl/
        ├── I<Module>.aidl
        ├── Android.bp
        ├── <Module>Hal.cpp
        ├── <Module>HalService.cpp
        └── service/
              └── android.hardware.<module>-service.rc

------------------------------------------------------------------------

## 2.4 AIDL HAL 架構圖（Binder IPC 全流程）

    App (Java/Kotlin)
       │
       ▼
    Framework Service (Java)
       │
       ▼ Binder IPC
    Native Service (C++)
       │
       ▼ Binder IPC
    AIDL HAL Client (Bp interface)
       │
       ▼ Binder Driver
    AIDL HAL Service (Bn interface)
       │
       ▼
    Vendor HAL implementation (C++)
       │
       ▼
    Kernel driver

------------------------------------------------------------------------

## 2.5 AIDL Stable Interface（核心特點）

### 2.5.1 InterfaceVersion --- API 版本

    @VintfStability
    @version 3
    interface IVibrator {
        void on();
        void off();
        void newMethod();
    }

查詢版本：

    adb shell dumpsys android.hardware.vibrator.IVibrator/default | grep InterfaceVersion

------------------------------------------------------------------------

### 2.5.2 InterfaceHash --- API Hash

確認介面內容是否被修改：

    adb shell dumpsys android.hardware.usb.IUsb/default | grep InterfaceHash

------------------------------------------------------------------------

### 2.5.3 API 演進規則（向後相容）

-   可以新增 API\
-   **不能移除 API**\
-   **不能更改既有 API 的參數/回傳值**

CTS/VTS 會針對這些規則檢查。

------------------------------------------------------------------------

## 2.6 AIDL HAL Service 啟動流程

### ① main() 註冊 HAL Service

``` cpp
int main() {
    auto vibrator = ndk::SharedRefBase::make<Vibrator>();

    const std::string instance =
        std::string() + IVibrator::descriptor + "/default";

    AServiceManager_addService(
        vibrator->asBinder().get(), instance.c_str());

    ABinderProcess_joinThreadPool();
}
```

------------------------------------------------------------------------

### ② init.rc 啟動 vendor HAL

    /vendor/etc/init/android.hardware.vibrator-service.rc

``` rc
service vendor.vibrator-hal /vendor/bin/hw/android.hardware.vibrator-service
    class hal
    user root
    group root
    seclabel u:r:hal_vibrator_default:s0
```

------------------------------------------------------------------------

### ③ ServiceManager 註冊服務

    adb shell service list | grep vibrator

------------------------------------------------------------------------

### ④ Framework 綁定 HAL

``` cpp
auto hal = IVibrator::fromBinder(
    ndk::SpAIBinder(AServiceManager_waitForService(
        "android.hardware.vibrator.IVibrator/default")));
```

------------------------------------------------------------------------

## 2.7 AIDL HAL IPC 呼叫範例（C++）

    App → Framework → Native Service → Binder → HAL

### IPC 呼叫：

``` cpp
sp<IVibrator> vibrator = ...;
vibrator->on(); // IPC
```

Binder 內部流程：

    BpVibrator::on() → Binder driver → BnVibrator::on() → Vibrator::on()

------------------------------------------------------------------------

## 2.8 AIDL VINTF 設定

### Framework manifest

    /system/etc/vintf/manifest.xml

``` xml
<hal format="aidl">
    <name>android.hardware.vibrator</name>
    <version>3</version>
</hal>
```

------------------------------------------------------------------------

### Vendor manifest

    /vendor/etc/vintf/manifest.xml

------------------------------------------------------------------------

📌 注意：manifest 的 version 只是「宣告」，真正跑的版本要用 dumpsys 查。

------------------------------------------------------------------------

## 2.9 如何查 AIDL HAL 實際版本（最準確）

    adb shell dumpsys android.hardware.<hal>.I<Interface>/default

例如：

    adb shell dumpsys android.hardware.power.IPower/default

輸出：

    InterfaceVersion: 3
    InterfaceHash: 4bc2aaab...

------------------------------------------------------------------------

## 2.10 Qualcomm 平台上的 AIDL HAL 常見位置

    /vendor/lib64/aidl/
    /vendor/bin/hw/
    /vendor/etc/init/
    /vendor/etc/vintf/

------------------------------------------------------------------------

## 2.11 AIDL HAL Debug 方法

### ✔ dumpsys（最常用）

    dumpsys android.hardware.usb.IUsb/default

### ✔ logcat

    logcat | grep -i hal

### ✔ 查看 HAL service 是否啟動

    ps -A | grep vibrator

### ✔ 檢查 Binder 狀態

    dumpsys binder

------------------------------------------------------------------------

## Chapter 2 小結

-   AIDL HAL 使用 Binder IPC\
-   Stable Interface = AIDL 的核心\
-   InterfaceVersion / Hash 決定 HAL 的穩定性\
-   HAL 透過 init.rc 啟動並經 ServiceManager 註冊\
-   真正跑的版本必須用 dumpsys 查詢\
-   Qualcomm AIDL HAL 主要位於 vendor 分區
