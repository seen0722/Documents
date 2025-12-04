# Part B · Chapter 8 --- HAL Crash / 啟動失敗排查完全指南

------------------------------------------------------------------------

## 8.1 HAL 啟動流程回顧（AIDL 與 HIDL）

### 🟩 AIDL HAL 啟動流程

    init.rc
        ↓
    啟動 /vendor/bin/hw/<service>
        ↓
    AServiceManager_addService()
        ↓
    註冊到 ServiceManager
        ↓
    ABinderProcess_joinThreadPool()

### 🟦 HIDL HAL 啟動流程

    init.rc
        ↓
    啟動 /vendor/bin/hw/<service>@<version>
        ↓
    registerAsService()
        ↓
    註冊到 hwservicemanager
        ↓
    加入 hwbinder thread loop

HAL 只要在任一階段出錯 → 會 crash 或無法啟動。

------------------------------------------------------------------------

# ⭐ 8.2 HAL 無法啟動時的完整排查流程

------------------------------------------------------------------------

## ✔ Step 1 --- 查 HAL 是否成功註冊

### AIDL：

    adb shell service list | grep hardware

### HIDL：

    adb shell lshal | grep hardware

若沒有出現 → HAL 尚未成功註冊。

------------------------------------------------------------------------

## ✔ Step 2 --- 查 init 啟動錯誤（最重要）

    adb logcat -b all | grep -i init

常見：

    init: cannot find '/vendor/bin/hw/android.hardware.power-service'
    init: Service crashed 4 times in 4 seconds, disabling
    init: sepolicy denied

意義： - binary 路徑錯誤 - 反覆 crash - SELinux 阻擋

------------------------------------------------------------------------

## ✔ Step 3 --- 查 HAL process 是否存在

    adb shell ps -A | grep hal

若 PID 不斷變動 → crash loop。

------------------------------------------------------------------------

## ✔ Step 4 --- 查 HAL crash log（logcat）

    adb logcat | grep -i hal
    adb logcat | grep -i binder
    adb logcat | grep -i ServiceManager

常見 crash：

### 🔥 SIGSEGV

    Fatal signal 11 (SIGSEGV)

原因：空指標、非法存取、driver 回傳錯誤。

### 🔥 Binder crash

    binder: transaction failed
    ServiceManager: Cannot register service

### 🔥 AIDL version mismatch

    Cannot find interface hash

------------------------------------------------------------------------

## ✔ Step 5 --- 查 kernel log（dmesg）

    adb shell dmesg | grep avc
    adb shell dmesg | grep binder
    adb shell dmesg | grep hal

常見：

### SEPolicy deny

    avc: denied { read write } for path=/dev/power_hal

### Driver ioctl 錯誤

    power_hal_ioctl: invalid argument

### Binder kernel 錯誤

    binder: transaction failed

------------------------------------------------------------------------

## ✔ Step 6 --- 用 strace 查 HAL 卡在哪

    adb shell strace -p $(pidof android.hardware.power-service)

可看到：

-   open("/dev/...")\
-   ioctl()\
-   binder syscall\
-   卡在某個 mutex

範例：

    ioctl(5, 0xc0186200, 0x7fc3e0f000) = -1 EINVAL

代表 ioctl 參數錯誤 → HAL / Driver mismatch。

------------------------------------------------------------------------

## ✔ Step 7 --- 查 VINTF mismatch

    adb shell dumpsys android.os.VintfService

可看到：

-   system manifest vs vendor manifest\
-   FCM version\
-   HAL version mismatch

------------------------------------------------------------------------

# ⭐ 8.3 常見 HAL Crash 類型（逐一解析）

------------------------------------------------------------------------

## 🔥 類型 A：ServiceManager/hwservicemanager 註冊失敗

AIDL：

    Cannot register android.hardware.power.IPower/default

HIDL：

    HIDL: Failed to register service

原因：

-   instance 名稱錯誤\
-   VINTF 不符\
-   binder thread pool 問題\
-   sepolicy 阻擋

------------------------------------------------------------------------

## 🔥 類型 B：SEPolicy 問題（最容易被忽略）

    avc: denied { read write } for path=/dev/xxx

必須修：

-   vendor.te\
-   hal\_`<module>`{=html}\_default.te\
-   file_contexts

------------------------------------------------------------------------

## 🔥 類型 C：找不到 driver node

    open("/dev/my_driver", O_RDWR) = -1 ENOENT

原因：

-   驅動沒建立 node\
-   uevent 未產生\
-   權限錯誤（0660 → 0600）

------------------------------------------------------------------------

## 🔥 類型 D：HAL 缺少函式庫（linker error）

    CANNOT LINK EXECUTABLE: cannot locate symbol ...

原因：

-   HAL 連到不存在的 vendor.so\
-   版本不符\
-   lib path 設錯

------------------------------------------------------------------------

## 🔥 類型 E：AIDL/HIDL 版本 mismatch（VINTF fail）

    InterfaceVersion mismatch
    InterfaceHash mismatch

原因：

-   system 要 v3\
-   vendor 實際提供 v2\
-   framework/vendor 不相容

此錯誤必定 crash。

------------------------------------------------------------------------

## 🔥 類型 F：AIDL Thread Pool 問題

    binder: transaction failed

原因：

-   沒呼叫 ABinderProcess_joinThreadPool()\
-   thread pool 數量不足

------------------------------------------------------------------------

## 🔥 類型 G：HIDL 未呼叫 registerAsService()

    HIDL: no instance registered

原因：

-   main() 寫錯\
-   early return\
-   service 實作未建立

------------------------------------------------------------------------

# ⭐ 8.4 Qualcomm 平台最常見 HAL Crash（實戰）

------------------------------------------------------------------------

## ⚡ Power HAL crash

    Fatal signal 11
    avc: denied /dev/kgsl-3d0

解法：

-   sepolicy\
-   驅動 node 權限\
-   vendor.qti.power.so 版本不符

------------------------------------------------------------------------

## ⚡ USB HAL 啟動錯誤

    UsbHal: UEventListener invalid

原因：

-   /sys/class/typec/ 資料夾異常

------------------------------------------------------------------------

## ⚡ Camera Provider crash（最常見的大魔王）

    CameraProvider@2.6 cannot load camera.qcom.so

可能原因：

-   camera.qcom.so 遺失\
-   HIDL provider 版本不符\
-   驅動/firmware 欠缺\
-   SELinux denial

------------------------------------------------------------------------

## ⚡ Vibrator HAL 錯誤

    Cannot find interface hash for IVibrator

原因：

-   AIDL/HIDL 混合\
-   vendor 舊 HAL 未移除

------------------------------------------------------------------------

# ✔ Chapter 8 小結

HAL crash / 啟動失敗完整調查路線：

1.  查 HAL 是否註冊（service / lshal）\
2.  查 init 啟動 log（最重要）\
3.  查 HAL 是否 crash loop（ps/top）\
4.  查 logcat（SIGSEGV / binder / linker）\
5.  查 dmesg（SEPolicy / ioctl / driver）\
6.  strace 找卡住的位置\
7.  查 VINTF mismatch

最常見錯誤：

-   SELinux deny\
-   driver node 問題\
-   HAL binary 缺少 so\
-   AIDL/HIDL 版本 mismatch\
-   thread pool 問題\
-   init.rc 與 service 名稱錯誤

掌握這一章，你可以 debug 95% 的 HAL 問題。
