# Part B · Chapter 10 --- HAL Debug Cheat Sheet（工程師口袋版）

------------------------------------------------------------------------

# ✔ 10.1 一眼辨識：這個 HAL 是 AIDL 還是 HIDL？

  判斷方式                     類型
  ---------------------------- --------------
  service 名稱沒有 @           **AIDL HAL**
  service 名稱含 @1.x / @2.x   **HIDL HAL**

範例：

    android.hardware.power.IPower/default        ← AIDL
    android.hardware.power@1.3::IPower/default  ← HIDL

------------------------------------------------------------------------

# ✔ 10.2 查 HAL 版本（最關鍵指令）

### 🟩 AIDL Version / Hash

    adb shell dumpsys android.hardware.<hal>.I<Interface>/default

### 🟦 HIDL Version

    adb shell lshal | grep <hal>

------------------------------------------------------------------------

# ✔ 10.3 HAL Debug 最常用指令合集

    # AIDL Service 列表
    service list | grep hardware

    # HIDL Service 列表
    lshal | grep hardware

    # 查 AIDL HAL 版本
    dumpsys android.hardware.<HAL>.I<Interface>/default

    # 查 HIDL HAL instance
    dumpsys hwservicemanager | grep -A5 <hal>

    # 查 HAL Process 是否存在
    ps -A | grep hal

    # 查 HAL Crash
    logcat | grep -i hal

    # 查 Driver / SELinux / Binder 錯誤
    dmesg | grep -i hal
    dmesg | grep avc
    dmesg | grep binder

    # 查 HAL 卡住點
    strace -p $(pidof <hal-process>)

    # 查 VINTF 契約
    dumpsys android.os.VintfService

------------------------------------------------------------------------

# ✔ 10.4 HAL 啟動失敗 Debug 流程（最常用）

                          HAL 無法啟動？
                                 │
             ┌───────────────────┴───────────────────┐
             │                                       │
    service / lshal 無註冊？                 有註冊但反覆 Crash？
             │                                       │
      查 init error                           查 logcat crash
    (logcat -b all | grep init)         (SIGSEGV / linker / binder)
             │                                       │
             └───────────────┐             ┌────────┘
                             │             │
                      查 SELinux deny？   查 Driver ioctl？
                        (dmesg | grep avc) (dmesg | grep <driver>)
                             │             │
                             └──────┐    ┌─┘
                                    │    │
                              修 sepolicy   修 driver / HAL mapping

------------------------------------------------------------------------

# ✔ 10.5 HAL Crash 原因速查表（95% 都是這些）

  類型                      錯誤訊息                   解法
  ------------------------- -------------------------- ------------------------------
  SELinux deny              `avc: denied`              加 vendor.te / file_contexts
  Driver node 不存在        `ENOENT`                   建立 /dev node / 修 uevent
  ioctl 錯誤                `EINVAL`                   修 ioctl 編號或參數
  linker 錯誤               `CANNOT LINK EXECUTABLE`   補 vendor.so
  AIDL/HIDL 版本 mismatch   `Expected version`         修 VINTF 或 HAL
  registerAsService 缺漏    HIDL 未註冊                修 main()
  joinThreadPool 缺漏       AIDL 無回應                加入 thread pool
  HAL 回傳假 success        CTS fail                   修 HAL 行為

------------------------------------------------------------------------

# ✔ 10.6 SELinux Debug 最快方法

    adb shell dmesg | grep avc

看到任何：

    avc: denied { read write } for ...

→ **100% 是 SEPolicy 問題**

要修改：

-   vendor/`<domain>`{=html}.te\
-   hal\_`<module>`{=html}\_default.te\
-   file_contexts\
-   genfs_contexts

------------------------------------------------------------------------

# ✔ 10.7 VINTF Mismatch Debug（最佳三步驟）

### Step 1：查 system 要求什麼版本

    /system/etc/vintf/compatibility_matrix.xml

### Step 2：查 vendor 實際提供什麼版本

    /vendor/etc/vintf/manifest.xml

### Step 3：查實際跑的 HAL 版本

    dumpsys android.hardware.<hal>.I<Interface>/default

➡ 三個不一致 → 必 fail。

------------------------------------------------------------------------

# ✔ 10.8 CTS FAIL 快速查找（最有效流程）

    grep android.hardware -R cts*

找出哪個 HAL fail。\
然後依 HAL 類型調查：

### AIDL：

    dumpsys android.hardware.<hal>.I<Interface>/default

### HIDL：

    lshal | grep <hal>

### 驅動：

    dmesg | grep <node>

### crash：

    logcat | grep -i hal

------------------------------------------------------------------------

# ✔ 10.9 Qualcomm HAL Debug 常用路徑

  HAL            資料夾位置
  -------------- --------------------------------------
  Power HAL      /vendor/lib64/hw/power.qti.so
  Camera HAL     /vendor/lib64/camera / vendor.qti.\*
  USB HAL        /vendor/lib64/hw/usb.qti.so
  Vibrator HAL   /vendor/lib64/hw/
  Lights HAL     /vendor/lib64/hw/lights.qti.so
  Health HAL     /vendor/lib64/hw/health.qti.so

這些路徑錯誤或版本不符 → 一定 crash。

------------------------------------------------------------------------

# ⭐ 10.10 HAL Debug 金三角（成功率 95%）

    1. dumpsys   ← 查 HAL 版本與行為
    2. logcat    ← 查 HAL crash
    3. dmesg     ← 查 driver / SELinux

掌握這三步驟，你就能解決 95% HAL 問題。

------------------------------------------------------------------------

# ✔ Chapter 10 小結

-   判斷 HAL 類型：有無 @\
-   判斷版本：AIDL 用 dumpsys；HIDL 用 lshal\
-   Crash：logcat\
-   Driver/Permission：dmesg\
-   卡住：strace\
-   不啟動：init log\
-   CTS FAIL → grep namespace + VINTF + Δ version\
-   Qualcomm → 多半是版本 mismatch 或 SELinux

這份 Cheat Sheet 是整本 HAL 手冊最精華的速查文件。
