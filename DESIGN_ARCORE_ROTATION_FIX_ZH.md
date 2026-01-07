# 設計文件：ARCore 90度旋轉架構對齊

| 欄位 | 詳情 |
| :--- | :--- |
| **專案** | Project Thorpe |
| **功能** | ARCore 支援與顯示方向對齊 |
| **狀態** | 已完成實作 / 已驗證 |
| **作者** | Android BSP 團隊 |

---

## 1. 執行摘要 (Executive Summary)
本文件概述了為解決 Project Thorpe 中 ARCore 應用程式出現 90 度影像旋轉錯誤所需的架構變更。解決方案的核心在於將架構從「強制橫屏」的系統屬性方案，轉向「以直屏為基準（Portrait-baseline）並配合框架補償」的對齊架構。

## 2. 背景與脈絡 (Context & Background)
### 2.1 硬體限制
本裝置選用的是 **原生直屏（Portrait-native）顯示面板**。在開發初期，為了符合產品的 **橫向（Landscape）使用定義**，系統被配置為將旋轉 90 度的狀態視為原生方向。

### 2.2 舊有實作方案
先前系統屬性設定如下：
`ro.surface_flinger.primary_display_orientation = ORIENTATION_90`

雖然這讓 UI 能在開機階段即以橫向顯示，但卻引入了與 Android 標準框架預期不符的根本矛盾，特別是針對依賴面板物理座標系統的組件。

## 3. 問題描述 (Problem Statement)
ARCore 應用程式在相機預覽與空間追蹤演算法中出現了持續性的 **90 度旋轉錯位**：
1.  **追蹤失敗**：感測器與相機座標的不匹配導致 ARCore 校準無法收斂。
2.  **UI/UX 異常**：在第三方 AR 應用中，相機預覽影像出現拉伸或方向錯誤。

## 4. 根因分析 (Root Cause Analysis)
根源在於系統重新定義的「自然方向（Natural Orientation）」與 ARCore 內部架構假設之間的 **核心邏輯衝突**：

1.  **組件層級的一致性**：在舊有實作中，**Camera HAL** 和 **Sensor HAL** 都被配置為符合 `ORIENTATION_90` 的定義。從單一組件的角度來看，它們回報的座標相對於系統定義的原生方向是「正確」的。
2.  **ARCore 內部假設**：然而，ARCore 內部的座標轉換流程假設裝置的 **物理基準方向**（即無系統旋轉時的方向）即為真正的 `ORIENTATION_0`。
3.  **方向的雙重映射 (Double-Mapping)**：
    *   透過設定 `ro.surface_flinger.primary_display_orientation = ORIENTATION_90`，系統向所有 App 提供了一個經過轉換的座標空間。
    *   ARCore 在解讀裝置屬性或顯示參數時，會在其認知中的「直屏面板」基礎上，額外疊加一層內部的旋轉處理。
    *   這導致了 **運行時方向錯誤（90度偏移）**，因為 ARCore 內部的 IMU/視覺融合邏輯無法與系統已「預先旋轉」的自然狀態調和。
    *   **簡而言之**：HAL 層雖與系統定義一致，但 **系統本身與 ARCore 對於原生直屏面板的底層預期不一致**。

## 5. 建議架構 (Proposed Architecture)
核心策略是將系統的「底層事實」回歸至面板的原生直屏模式，隨後在框架（Framework）與 HAL 層級處理橫屏需求。

### 5.1 系統屬性對齊
將 SurfaceFlinger 方向重置為原生基準：
- `ro.surface_flinger.primary_display_orientation = ORIENTATION_0`

### 5.2 HAL 層級重整 (HAL Layer Reconstruction)
為了在基準為直屏的情況下維持橫向使用體驗，必須重新對齊 HAL：
*   **感測器 (Sensor HAL)**：實作座標對換 (X ↔ Y) 與正負號調整，確保感測器融合能正確感知相對於橫向產品設計的「上方」。
*   **相機 (Camera HAL)**：
    *   更新 `sensorOrientation`（通常根據安裝方式調整為 270 或 90）以匹配新的系統基準。
    *   調整元數據 (Metadata)，確保影像緩衝區在傳輸至框架前已由硬體/驅動層正確旋轉。
*   **觸控輸入 (Touch Input HAL)**：重新映射觸控座標以匹配顯示變換，確保觸點與視覺精確對齊。

### 5.3 框架補償 (Framework Compensation)
確保裝置在使用者介面（UI）層級預設為橫屏：
*   **視窗管理 (WMS)**：修改 `DisplayRotation.java`，將預設旋轉設為 `ROTATION_90`。
*   **設定提供者 (Settings Provider)**：確保 `def_user_rotation` 初始化為 1 (Landscape)。
*   **系統 UI 與應用**：最佳化開機動畫 (Boot Animation)、啟動器 (Launcher) 與設定精靈 (SUW)，使其預設以橫向顯示。

## 6. 驗證計劃
修復方案經過三個迭代階段驗證：
1.  **功能驗證**：確認 ARCore 追蹤狀態呈現為「健康（Healthy）」。
2.  **迴歸測試**：
    *   相機 UI 在垂直/橫向切換時的表現。
    *   最近使用的程式（Overview）縮圖算繪。
    *   設定精靈（SUW）在各種旋轉鎖定下的佈局。
3.  **第三方相容性**：驗證 QR 碼掃描器與照片編輯器（如 Google 相簿、Instagram）。

## 8. 實作細節 (代碼參考)
本節提供實作架構對齊時的完整代碼變更 (Diffs)，便於工程端直接查閱。

### 8.1 系統配置
**Patch:** `build/make/0001-Remove-surface_flinger-property.patch`
```bash
--- a/tools/buildinfo_odm.sh
+++ b/tools/buildinfo_odm.sh
-echo "ro.surface_flinger.primary_display_orientation=ORIENTATION_90"
```

**Patch:** `frameworks/base/0002-Update-BSP-...-defaults.xml`
```xml
--- a/packages/SettingsProvider/res/values/defaults.xml
+++ b/packages/SettingsProvider/res/values/defaults.xml
-    <integer name="def_user_rotation">0</integer>
+    <integer name="def_user_rotation">1</integer>
```

### 8.2 HAL 層級對齊
#### 觸控輸入 (核心驅動)
**Patch:** `kernel/msm-5.4/0001-UPDATE-BSP-Display-Rotation-Revert-Touch-rotation.patch`
```c
--- a/drivers/input/touchscreen/egalax_i2c.c
+++ b/drivers/input/touchscreen/egalax_i2c.c
-//#define _SWITCH_XY
+#define _SWITCH_XY
 //#define _CONVERT_Y
-//#define _CONVERT_X
+#define _CONVERT_X
```

#### 感測器 HAL (Registry 配置)
**Patch:** `vendor/qcom/proprietary/0002-UPDATE-BSP-Display-Rotation-Change-g-sensor-gyro-ori.patch`
```json
--- a/sensors-see/registry/config/lahaina/kodiak_idp_lsm6dst_0.json
+++ b/sensors-see/registry/config/lahaina/kodiak_idp_lsm6dst_0.json
     ".orient":{
-      "x":{ "data": "+x" },
-      "y":{ "data": "-y" },
-      "z":{ "data": "-z" }
+      "x":{ "data": "-y" },
+      "y":{ "data": "-x" },
+      "z":{ "data": "-z" }
     }
```

#### 相機 (裝置樹)
**Patch:** `vendor/qcom/proprietary/0001-UPDATE-BSP-Display-Rotation-Change-camera-sensor-ori.patch`
```dts
--- a/camera-devicetree/yupik-camera-sensor-idp.dtsi
+++ b/camera-devicetree/yupik-camera-sensor-idp.dtsi
-		sensor-position-roll = <0>;
+		sensor-position-roll = <90>;
```

### 8.3 框架與 UI 補償
#### 開機動畫
**Patch:** `frameworks/base/0001-UPDATE-BSP-Display-Rotation-Change-boot-animation-or.patch`
```cpp
--- a/cmds/bootanimation/BootAnimation.cpp
+++ b/cmds/bootanimation/BootAnimation.cpp
+    SurfaceComposerClient::Transaction t;
+    int d_width = resolution.getWidth(), d_height = resolution.getHeight();
+    d_width = resolution.getHeight();
+    d_height = resolution.getWidth();
+    Rect destRect(d_width, d_height);
+    t.setDisplayProjection(mDisplayToken, ui::ROTATION_90, destRect, destRect);
```

#### 視窗管理服務 (DisplayRotation)
**Patch:** `frameworks/base/0002-Update-BSP-Display-Rotation-Fixed-display-flipping-i.patch`
```java
--- a/services/core/java/com/android/server/wm/DisplayRotation.java
+++ b/services/core/java/com/android/server/wm/DisplayRotation.java
+        // 客製化偏移：僅針對內部預設面板增加 +90°
+        if(displayContent.isDefaultDisplay && (displayContent.mDisplay.getType() == TYPE_INTERNAL))
+        {
+		    return Surface.ROTATION_90;
+        }
```

#### Launcher3
**Patch:** `packages/apps/Launcher3/0001-Update-BSP-Display-Rotation-Fixed-display-flipping-i.patch`
```java
--- a/src/com/android/launcher3/states/RotationHelper.java
+++ b/src/com/android/launcher3/states/RotationHelper.java
-            activityFlags = SCREEN_ORIENTATION_NOSENSOR;
+            activityFlags = SCREEN_ORIENTATION_LANDSCAPE;
```

#### Launcher3 最近使用程式 (縮圖方向修正)
**Patch:** `packages/apps/Launcher3/0002-BUG-FIX-BSP-Display-Rotation-Fix-issues-about-the-or.patch`
```java
--- a/quickstep/src/com/android/quickstep/util/RecentsOrientedState.java
+++ b/quickstep/src/com/android/quickstep/util/RecentsOrientedState.java
-    private @SurfaceRotation int mRecentsActivityRotation = ROTATION_0;
+    private @SurfaceRotation int mRecentsActivityRotation = ROTATION_90;
     // ...
-            return ROTATION_0;
+            return ROTATION_90;
```

#### 框架 (啟用四向旋轉)
**Patch:** `frameworks/base/0003-BUG-FIX-BSP-Display-Rotation-Fix-the-issue-that-auto.patch`
```xml
--- a/core/res/res/values/config.xml
+++ b/core/res/res/values/config.xml
-    <bool name="config_allowAllRotations">false</bool>
+    <bool name="config_allowAllRotations">true</bool>
```

#### GMS 設定精靈 (SUW)
**Patch:** `vendor/partner_gms/0001-Update-BSP-Display-Rotation-Fixed-display-flipping-i.patch`
```makefile
--- a/products/gms.mk
+++ b/products/gms.mk
-    ro.setupwizard.rotation_locked=true \
+    ro.setupwizard.rotation_locked=false \
```

### 8.4 應用層級修正
#### SnapdragonCamera (圖示旋轉補償)
**Patch:** `vendor/codeaurora/packages/apps/SnapdragonCamera/0001-UPDATE-AP-DISPLAY-Rotation-Rotate-90-degree-for-came.patch`
```java
--- a/src/com/android/camera/CameraActivity.java
+++ b/src/com/android/camera/CameraActivity.java
+            // thorpe: 因 ARCore Patch 需要，將方向補償 90 度
+            orientation = (orientation + 90) % 360;
// ...
--- a/src/com/android/camera/util/CameraUtil.java
+++ b/src/com/android/camera/util/CameraUtil.java
+            // thorpe: 因 ARCore Patch 需要，JPEG 旋轉減 90 度
+            orientation = (orientation - 90) % 360;
```

#### SnapdragonCamera (橫向鎖定)
**Patch:** `vendor/codeaurora/packages/apps/SnapdragonCamera/0002-UPDATE-AP-DISPLAY-Rotation-Rotate-90-degree-for-came.patch`
```xml
--- a/AndroidManifest.xml
+++ b/AndroidManifest.xml
-            android:screenOrientation="unspecified"
+            android:screenOrientation="landscape"
```
```java
--- a/src/com/android/camera/CameraActivity.java
+++ b/src/com/android/camera/CameraActivity.java
+        Log.d(TAG, "Set orientation default to landscape.");
+        setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_LANDSCAPE);
```

### 8.5 感測器校正微調
#### 電子羅盤校正矩陣
**Patch:** `vendor/qcom/proprietary/0004-UPDATE-BSP-E-COMPASS-fine-tune-ist8306-E-compass-cor.patch`
```json
--- a/sensors-see/registry/config/lahaina/kodiak_idp_ist8306_0.json
+++ b/sensors-see/registry/config/lahaina/kodiak_idp_ist8306_0.json
     ".corr_mat":{
-          "0_0":{ "data": "0.93875351" },
+          "0_0":{ "data": "1.006126579" },
          // ... (完整校正矩陣已依旋轉後方向重新校準)
     }
```

## 9. 結論
透過將架構重新對齊為 **「直屏基準 + 預設橫屏 (Portrait Baseline + Landscape Default)」** 模式，我們成功解決了 ARCore 90 度旋轉問題，同時完整保留了產品的橫向特色。這確保了裝置與 Android 生態系統的全面相容性，並支援未來的 ARCore 更新。
