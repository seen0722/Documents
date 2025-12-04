# Part B · Chapter 7 --- HAL 版本與狀態檢查技巧（AIDL / HIDL 實務指南）

------------------------------------------------------------------------

## 7.1 為什麼要查 HAL 的「實際版本」？

Android 專案中常見情況：

-   manifest 宣告的版本\
-   驅動或 SoC SDK 提供的版本\
-   裝置上真正跑的版本

三者常常不同。

因此判斷 HAL「實際跑哪一版」是 debug 的核心。

------------------------------------------------------------------------

# 🟩 7.2 如何判斷 HAL 是 AIDL 還是 HIDL？

最簡單規則：

### ✔ AIDL HAL（無 @）

    android.hardware.power.IPower/default
    android.hardware.usb.IUsb/default
    android.hardware.vibrator.IVibrator/default

### ✔ HIDL HAL（含 @version）

    android.hardware.power@1.3::IPower/default
    android.hardware.camera.provider@2.6::ICameraProvider/legacy/0

規則：\
**有 @ = HIDL**\
**沒有 @ = AIDL**

------------------------------------------------------------------------

# 🟦 7.3 查 AIDL HAL 實際版本（最準確方法）

使用 dumpsys：

    adb shell dumpsys android.hardware.<hal>.I<Interface>/default

例如：

    adb shell dumpsys android.hardware.power.IPower/default

輸出包含：

    InterfaceVersion: 3
    InterfaceHash: 4bc2aaab...

含義：

  欄位               說明
  ------------------ -----------------------------
  InterfaceVersion   AIDL HAL API 版本（最重要）
  InterfaceHash      API 簽名的 hash

------------------------------------------------------------------------

# 🟥 7.4 查 HIDL HAL 實際版本（最準確：lshal）

    adb shell lshal | grep <hal>

範例：

    android.hardware.power@1.3::IPower/default

可得知：

-   HAL 版本 = 1.3\
-   instance = default\
-   transport = binderized / passthrough

也可使用：

    adb shell dumpsys hwservicemanager

------------------------------------------------------------------------

# 🟧 7.5 查 HAL transport（AIDL / HIDL 均可）

AIDL：\
永遠使用 Binder，不存在 passthrough。

HIDL：\
使用 lshal：

    adb shell lshal | grep -A 3 <hal>

可能看到：

    Transport: hwbinder

或：

    Transport: passthrough (arch: 64)

------------------------------------------------------------------------

# 🟫 7.6 HAL 是否正常運作？（完整 checklist）

------------------------------------------------------------------------

## ✔ 1. 是否註冊成功？

AIDL：

    service list | grep hardware

HIDL：

    lshal | grep hardware

------------------------------------------------------------------------

## ✔ 2. HAL service 是否正在運作？

    ps -A | grep power

------------------------------------------------------------------------

## ✔ 3. 是否反覆 crash？

    logcat | grep -i hal

常見錯誤：

    Service crashed 4 times in 4 seconds, disabling

------------------------------------------------------------------------

## ✔ 4. 是否有 SELinux deny？

    dmesg | grep avc

------------------------------------------------------------------------

## ✔ 5. HAL 是否成功開啟 driver？

    strace -p <pid>

會顯示：

-   open("/dev/...")\
-   ioctl()

------------------------------------------------------------------------

## ✔ 6. 是否符合 VINTF 契約？

    dumpsys android.os.VintfService

可看到：

-   manifest vs vendor\
-   FCM version\
-   mismatch 詳細資訊

------------------------------------------------------------------------

# 🟪 7.7 查 HAL instance 名稱

AIDL：

    service list

HIDL：

    lshal

範例：

    android.hardware.camera.provider@2.7::ICameraProvider/legacy/0

instance = `"legacy/0"`

------------------------------------------------------------------------

# 🟧 7.8 Qualcomm 平台 HAL 版本檢查技巧

查看 HAL .so：

    ls /vendor/lib64/hw/android.hardware.*

可以識別：

-   Qualcomm HAL\
-   Google AOSP HAL\
-   ODM 自己寫的 HAL

------------------------------------------------------------------------

# ⭐ 7.9 Debug FAQ（工程師最常遇到的狀況）

------------------------------------------------------------------------

### Q1：manifest version = 3，但 dumpsys 顯示 version = 2？

A：\
manifest 宣告 ≠ 實際執行版本。\
真正版本以 dumpsys 為準。

------------------------------------------------------------------------

### Q2：AIDL 和 HIDL HAL 同時存在，正常嗎？

A：\
Android 過渡期（12\~14）會同時存在。

------------------------------------------------------------------------

### Q3：HAL 執行後立刻退出？

A：\
大多是 SEPolicy 問題 → `dmesg | grep avc`

------------------------------------------------------------------------

### Q4：HAL 卡住？

A：\
用 `strace` 看是否卡在 driver ioctl。

------------------------------------------------------------------------

# ✔ Chapter 7 小結

-   **AIDL HAL 版本：使用 dumpsys 查 InterfaceVersion / Hash**\
-   **HIDL HAL 版本：使用 lshal 查 @versions**\
-   \*\*是否是 AIDL 或 HIDL：看是否有 @*\*\
-   **HAL 正常與否：service/lshal + ps + logcat/dmesg**\
-   **HAL crash：logcat + dmesg（SELinux）**\
-   **HAL 卡住：strace**\
-   **VINTF 問題：dumpsys android.os.VintfService**

這些技巧能解決 90% 的 HAL 版本與狀態問題。
