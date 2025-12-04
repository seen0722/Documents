# Part A · Chapter 1 --- Android HAL 基礎概念

## 1.1 Android 系統架構（從上而下）

Android 軟體架構主要分為五層：

    ┌──────────────────────────┐
    │   Apps (Java/Kotlin)     │
    └───────────────▲──────────┘
                    │
    ┌───────────────┴──────────┐
    │   Framework (Java/Kotlin) │
    └───────────────▲──────────┘
                    │  Binder IPC
    ┌───────────────┴──────────┐
    │   Native services (C++)   │
    └───────────────▲──────────┘
                    │
    ┌───────────────┴──────────┐
    │   HAL (AIDL / HIDL / C++) │
    └───────────────▲──────────┘
                    │
    ┌───────────────┴──────────┐
    │   Kernel drivers (C)      │
    └──────────────────────────┘

HAL（Hardware Abstraction Layer）負責： - 讓 Framework 與硬體驅動分離 -
維持跨硬體平台一致的系統運作行為 - 透過 IPC 與 Native Service/Framework
溝通

------------------------------------------------------------------------

## 1.2 HAL 的本質：以 IPC 連結 Framework 與 Driver

HAL 不是 driver，而是 Framework 與 driver 中間的溝通層。

    App 
      ↓
    Framework Service 
      ↓ Binder IPC
    HAL (AIDL/HIDL/C++) 
      ↓
    Kernel driver (/dev/... , ioctl)

------------------------------------------------------------------------

## 1.3 HAL 存在的原因（設計理念）

### 1. 解決不同硬體平台的差異

讓 Framework 呼叫固定的 API，由 HAL 轉換為各廠商的硬體行為。

### 2. 實現 Project Treble（system/vendor 分離）

Android 8 之後 system 與 vendor 必須分離，HAL 是邊界。

### 3. 避免系統升級導致硬體失效

AIDL/HIDL 版本控管可保證向後相容。

------------------------------------------------------------------------

## 1.4 HAL 的三大類型（演進）

### Legacy HAL（Android 4--7）

-   `.so` 直接呼叫
-   沒有 IPC、沒有版本控制
-   與 Framework 強耦合，已被棄用

### HIDL HAL（Android 8--10）

-   使用 HwBinder IPC
-   名稱帶有 `@1.3::IFoo/default`
-   支援 Treble，但語法複雜，已逐步淘汰

### AIDL HAL（Android 11+）

-   使用 Binder IPC（與 Framework 同技術）
-   名稱無版本號：`android.hardware.xxx.Iyyy/default`
-   有 stable interface：InterfaceVersion / Hash
-   Google 正全面改為 AIDL

------------------------------------------------------------------------

## 1.5 Binder / HwBinder 的差異

  技術       用在       說明
  ---------- ---------- ---------------------------
  Binder     AIDL HAL   Framework 同一套
  HwBinder   HIDL HAL   為 HAL 特製的 binder 版本
  無 IPC     Legacy     直接 link library

------------------------------------------------------------------------

## 1.6 為什麼要淘汰 HIDL 改 AIDL？

-   HIDL 太複雜\
-   AIDL 版本升級更安全\
-   AIDL 支援 Java/Kotlin 呼叫\
-   CTS 對 AIDL 的檢查比較完整\
-   Google 官方強推 AIDL（已成為 HAL 主流）

------------------------------------------------------------------------

## 1.7 HAL 在 Project Treble 的角色

Treble 的關鍵目標：

    system (Google)
    ↕ HAL boundary
    vendor (OEM / SoC)

HAL 是 system/vendor 的通訊契約，並透過： - VINTF（manifest + matrix） -
AIDL/HIDL version 控制

確保 Android 升級不會破壞 vendor 分區。

------------------------------------------------------------------------

## Chapter 1 小結

-   HAL 是 Framework 與 Driver 中間的抽象層\
-   三代 HAL：Legacy → HIDL → AIDL（主流）\
-   AIDL HAL 擁有最佳版控、相容性、維護性
-   HAL 是 Treble 的核心
