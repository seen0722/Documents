# Part B · Chapter 9 --- CTS / VTS HAL Fail 分析與解法大全

------------------------------------------------------------------------

## 9.1 HAL 與 CTS/VTS 的關係

Android CTS/VTS 會驗證 HAL 的：

-   API 行為是否符合 AOSP 規範\
-   AIDL/HIDL IPC 是否正常\
-   HAL 版本是否符合 VINTF\
-   HAL 與 Kernel Driver 是否一致\
-   SELinux 權限是否正確\
-   效能（Camera/Power 等）\
-   系統整合行為

**HAL 行為錯誤、版本不符、SEPolicy 阻擋 → 必 fail。**

------------------------------------------------------------------------

# ⭐ 9.2 如何從 CTS/VTS log 找 HAL Fail

### ✔ Step 1 --- 搜尋 HAL namespace

    grep -R "android.hardware" -n cts*

### ✔ Step 2 --- 搜尋版本問題

    grep -R "InterfaceVersion" -n .
    grep -R "InterfaceHash" -n .

例：

    Expected version 3 but HAL reported version 2

→ **100% VINTF mismatch**。

------------------------------------------------------------------------

# ⭐ 9.3 HAL Fail 常見分類

  類型         說明                      常見 HAL
  ------------ ------------------------- -------------------------
  版本不符     HAL version \< CTS 期望   Power / Vibrator / USB
  回傳錯誤值   metadata or status 失常   Power / Camera / Health
  行為錯誤     HAL 回 OK 但行為不對      Lights / Vibrator
  Timeout      HAL 不回應                Camera / USB
  Crash        HAL segfault              Camera / Power
  SELinux      avc denied 導致 fail      所有 HAL

------------------------------------------------------------------------

# #️⃣ 9.4 Power HAL CTS Fail（非常常見）

------------------------------------------------------------------------

## ❌ FAIL 1：Power HAL version mismatch

    Expected IPower version 3 but got 2

原因：

-   Vendor HAL 舊版\
-   VINTF manifest 與 FCM 不符

解法：

-   更新 vendor power HAL\
-   修 VINTF\
-   對齊 framework version

------------------------------------------------------------------------

## ❌ FAIL 2：PowerHint 行為異常

例：

    Power HAL did not respond to LAUNCH hint

原因：

-   QTI Power HAL 未正確呼叫 boost node\
-   /sys/power node 不存在\
-   mapping 錯誤

------------------------------------------------------------------------

## ❌ FAIL 3：低電模式行為不符

    Device failed to enter low power mode

原因：

-   CPU online/offline 錯誤\
-   vendor HAL 寫死 return OK\
-   驅動沒有行為

------------------------------------------------------------------------

# #️⃣ 9.5 USB HAL CTS Fail

------------------------------------------------------------------------

## ❌ FAIL：Port role 錯誤

    Usb Hal failed to report current port role

原因：

-   Type-C 驅動不完整\
-   HAL 對 uevent 解析錯誤

------------------------------------------------------------------------

## ❌ FAIL：Data role 不符

    expected host mode but device reports device mode

解法：

-   修 typec 驅動\
-   修 HAL role mapping

------------------------------------------------------------------------

# #️⃣ 9.6 Camera HAL CTS Fail（最大宗）

------------------------------------------------------------------------

## ❌ FAIL：CameraProvider 啟動失敗

    Cannot connect to camera provider

原因：

-   camera HAL .so crash\
-   sensor 驅動錯誤\
-   provider 版本不符\
-   VINTF mismatch

------------------------------------------------------------------------

## ❌ FAIL：Camera metadata 錯誤

    Required field missing: android.lens.info.minimumFocusDistance

原因：\
metadata 填錯 or 缺少。

------------------------------------------------------------------------

## ❌ FAIL：Timeout 問題

    Camera device open time out

原因：

-   ISP firmware 問題\
-   驅動卡住

------------------------------------------------------------------------

# #️⃣ 9.7 Vibrator HAL CTS Fail

------------------------------------------------------------------------

## ❌ FAIL：version mismatch

    Expected IVibrator version 2 but HAL reports version 1

原因：

-   AIDL/HIDL 混用\
-   舊 HAL 未移除

------------------------------------------------------------------------

## ❌ FAIL：振動行為不符

    Device failed to vibrate with amplitude control

原因：

-   驅動不支援 amplitude\
-   HAL 卻回 OK（必 fail）

------------------------------------------------------------------------

# #️⃣ 9.8 Lights HAL CTS Fail

------------------------------------------------------------------------

## ❌ FAIL：亮度值錯誤

    Light brightness must be in range 0-255

原因：

-   HAL 回傳固定值\
-   驅動 mapping 錯誤

------------------------------------------------------------------------

# #️⃣ 9.9 Health（電池）HAL CTS Fail

------------------------------------------------------------------------

## ❌ FAIL：電量資訊錯誤

    Battery level must be between 0 and 100

原因：

-   驅動回錯誤值\
-   HAL mapping 錯誤

------------------------------------------------------------------------

# ⭐ 9.10 系統化定位 HAL CTS Fail（超實用）

------------------------------------------------------------------------

### ✔ Step 1 --- 找出 HAL namespace

    grep android.hardware -R .

### ✔ Step 2 --- 查 AIDL 版本與 hash

    dumpsys android.hardware.power.IPower/default

### ✔ Step 3 --- 用 lshal 查 HIDL 版本

    lshal | grep provider

### ✔ Step 4 --- 查 crash（logcat）

### ✔ Step 5 --- 查 driver 與 SELinux（dmesg）

### ✔ Step 6 --- strace 分析 HAL 行為

### ✔ Step 7 --- VINTF compliance 檢查

    dumpsys android.os.VintfService

------------------------------------------------------------------------

# ⭐ 9.11 Qualcomm 平台 HAL CTS Fail 總整理

-   Power HAL 舊版 → version mismatch\
-   USB HAL typec 驅動 mapping 錯\
-   Camera HAL provider 版本不符\
-   Vibrator HAL AIDL hash 錯誤\
-   Lights HAL 回 OK 但行為錯\
-   Health HAL 驅動回傳錯誤值\
-   SELinux deny 導致 HAL 無法操作 driver

------------------------------------------------------------------------

# ✔ Chapter 9 小結

-   HAL CTS Fail 主要來自：**版本不符、行為錯誤、driver
    mapping、SEPolicy**\
-   AIDL/HIDL mismatch 是最常見 fail\
-   Camera 是 fail 最多的 HAL\
-   Qualcomm HAL 版本眾多 → 最容易失配\
-   dumpsys + lshal + VINTF + logcat + dmesg 是找出 root cause
    的必備組合

掌握本章技巧，你能解決 95% HAL CTS 問題。
