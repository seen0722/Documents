# Part B · Chapter 6 --- HAL Debug 工具大全（完整工程師指南）

------------------------------------------------------------------------

## 6.1 HAL Debug 工具總覽

HAL Debug 工具可分為四大類：

    1. IPC / Service：service、lshal、binderctl
    2. Framework / HAL call flow：dumpsys
    3. 日誌：logcat、dmesg（kernel log）
    4. 系統偵錯：ps、top、strace、init log

------------------------------------------------------------------------

# 🟩 6.2 service list（AIDL HAL 專用）

列出所有 AIDL-based HAL service：

    adb shell service list | grep hardware

典型輸出：

    android.hardware.power.IPower/default
    android.hardware.usb.IUsb/default

用途：

-   ✔ 查 AIDL HAL 是否註冊成功\
-   ✔ 查 instance 名稱\
-   ✘ 無法查 HIDL HAL

------------------------------------------------------------------------

# 🟦 6.3 lshal（HIDL HAL 專用）

HIDL 的主要工具。

    adb shell lshal

可列出：

-   Binderized HAL\
-   Passthrough HAL\
-   VINTF manifest 內容\
-   HAL instance 列表

搜尋特定 HAL：

    adb shell lshal | grep camera

------------------------------------------------------------------------

# 🟥 6.4 dumpsys（最強大的 HAL Debug 工具）

------------------------------------------------------------------------

## 6.4.1 dumpsys 查 AIDL HAL 版本

    adb shell dumpsys android.hardware.power.IPower/default

輸出：

    InterfaceVersion: 3
    InterfaceHash: 4bc2aaab...

可用來查：

-   ✔ AIDL HAL 版本\
-   ✔ AIDL hash\
-   ✔ service 內部狀態

------------------------------------------------------------------------

## 6.4.2 dumpsys hwservicemanager 查 HIDL HAL

    adb shell dumpsys hwservicemanager

可看到：

-   HAL instance\
-   transport\
-   pid\
-   client 列表

------------------------------------------------------------------------

# 🟧 6.5 binderctl（部分裝置有）

列出 binder service：

    adb shell binderctl list

------------------------------------------------------------------------

# 🟨 6.6 logcat（排查 HAL crash 最重要工具）

    adb logcat | grep -i hal

常見錯誤：

    E android.hardware.power-service: Service crashed due to SIGSEGV
    E ServiceManager: Cannot register android.hardware.power.IPower/default

也能看到：

-   binder remote died\
-   SEPolicy deny\
-   服務反覆重啟

------------------------------------------------------------------------

# 🟫 6.7 dmesg（kernel log + binder kernel message）

    adb shell dmesg | grep -i binder

用途：

-   ✔ driver 問題\
-   ✔ ioctl 失敗\
-   ✔ binder kernel 錯誤\
-   ✔ selinux deny

------------------------------------------------------------------------

# 🟪 6.8 ps / top（查 HAL 是否在跑）

    adb shell ps -A | grep hal

用途：

-   ✔ HAL 是否啟動\
-   ✔ 是否一直重啟（restart loop）\
-   ✔ 是否 CPU 100%（死 loop）

------------------------------------------------------------------------

# 🟩 6.9 strace（分析 HAL 行為）

追蹤 HAL 系統呼叫：

    adb shell strace -p $(pidof android.hardware.power-service)

可看到：

-   open /dev node\
-   ioctl\
-   binder syscall\
-   HAL 卡在哪

------------------------------------------------------------------------

# 🟦 6.10 init log（HAL 啟動失敗第一線 clue）

    adb logcat -b all | grep -i init

常見：

    init: cannot find '/vendor/bin/hw/android.hardware.power-service'
    init: Service crashed 4 times in 4 seconds, disabling
    init: sepolicy denied

------------------------------------------------------------------------

# 🟥 6.11 SELinux deny（HAL 啟動失敗 90% 原因）

    adb shell dmesg | grep avc

錯誤：

    avc: denied { read write } for pid=3456 path=/dev/power_hal

代表須修改：

-   vendor.te\
-   file_contexts

------------------------------------------------------------------------

# 🟪 6.12 HAL Debug 工具比較表

  工具                         用途                AIDL   HIDL
  ---------------------------- ------------------- ------ ------
  service list                 查 AIDL HAL         ✔      ✘
  lshal                        查 HIDL HAL         ✘      ✔
  dumpsys `<service>`{=html}   查 AIDL 版本/hash   ✔      ✘
  dumpsys hwservicemanager     查 HIDL instance    ✘      ✔
  logcat                       查 crash            ✔      ✔
  dmesg                        查 kernel/binder    ✔      ✔
  ps/top                       查進程              ✔      ✔
  strace                       行為分析            ✔      ✔
  vintf dumps                  VINTF 檢查          ✔      ✔

------------------------------------------------------------------------

# ✔ Chapter 6 小結

-   AIDL → `service list`, `dumpsys`
-   HIDL → `lshal`, `dumpsys hwservicemanager`
-   HAL crash → `logcat`, `dmesg`, `init`
-   HAL 啟動失敗 → sepolicy、init.rc
-   HAL 卡住 → strace
-   VINTF 問題 → `dumpsys android.os.VintfService`

掌握這些工具即可 Debug 90% HAL 問題。
