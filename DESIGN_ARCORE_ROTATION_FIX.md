# Design Document: ARCore 90-Degree Rotation Architecture Alignment

| Field | Detail |
| :--- | :--- |
| **Project** | Project Thorpe |
| **Feature** | ARCore Support & Display Orientation Alignment |
| **Status** | Implementation Complete / Verified |
| **Author** | Android BSP Team |

---

## 1. Executive Summary
This document outlines the architectural changes required to resolve the 90-degree image rotation error in ARCore applications for Project Thorpe. The resolution involves pivoting from a "forced landscape" system property approach to a "portrait-baseline with framework compensation" architecture.

## 2. Context & Background
### 2.1 Hardware Constraints
The device utilizes a **Portrait-native Display Panel**. In the early development stages, the system was configured to treat the 90-degree rotated state as the primary orientation to meet the product's **Landscape-mode requirement**.

### 2.2 Legacy Implementation
Previously, the system property was set as:
`ro.surface_flinger.primary_display_orientation = ORIENTATION_90`

While this allowed the UI to appear in landscape mode from the early boot stages, it introduced a fundamental mismatch with Android's standard framework expectations, particularly for components that rely on the panel's physical coordinate system.

## 3. Problem Statement
ARCore applications exhibited a persistent **90-degree rotation mismatch** in the camera preview and spatial tracking algorithms. This led to:
1.  **Tracking Failure**: ARCore calibration could not converge due to the sensor/camera coordinate mismatch.
2.  **UI/UX Regressions**: Camera previews were stretched or incorrectly oriented in third-party AR apps.

## 4. Root Cause Analysis
The root cause is a **Core Logic Conflict** between the system's redefined "Natural Orientation" and ARCore's internal architectural assumptions:

1.  **Component-Level Consistency**: In the legacy implementation, both **Camera HAL** and **Sensor HAL** were configured to align with the `ORIENTATION_90` definition. From their individual perspectives, the coordinates being reported were "correct" relative to the system's defined primary orientation.
2.  **ARCore Internal Assumption**: However, ARCore's internal coordinate transformation pipeline assumes that the device's **Physical Baseline** (the orientation with no system-imposed rotation) is the true `ORIENTATION_0`. 
3.  **The Orientation Double-Mapping**:
    *   By setting `ro.surface_flinger.primary_display_orientation = ORIENTATION_90`, the system provided a transformed coordinate space to all applications.
    *   ARCore, interpreting the device property or display metric, applied its own set of internal rotations on top of what it perceived should be a portrait panel.
    *   This resulted in a **runtime orientation error** (90-degree offset) because ARCore's internal mapping logic for IMU/Vision fusion could not reconcile the fact that the system "Natural" state was already physically rotated.
    *   In short: The HALs were consistent with the system, but the **system itself was inconsistent with ARCore's hardcoded expectations** for portrait-native panels.

## 5. Proposed Architecture (The Alignment)
The core strategy is to return the system's "Ground Truth" to the panel's native portrait mode and then handle the landscape requirement at the Framework and HAL layers.

### 5.1 System Property Alignment
Revert the SurfaceFlinger orientation to the native baseline:
- `ro.surface_flinger.primary_display_orientation = ORIENTATION_0`

### 5.2 HAL Layer Reconstruction
To maintain the landscape usage while the baseline is portrait, the HALs must be re-aligned:
*   **Sensor HAL**: Implement a coordinate swap (X ↔ Y) and sign adjustment to ensure sensor fusion correctly perceives "Up" relative to the landscape product design.
*   **Camera HAL**: 
    *   Update `sensorOrientation` (typically to 270 or 90 depending on mounting) to match the new system baseline.
    *   Adjust metadata to ensure preview buffers are rotated correctly by the hardware/driver before reaching the framework.
*   **Touch Input HAL**: Re-map touch coordinates to match the display transformation, ensuring touch-to-visual precision.

### 5.3 Framework Compensation
Ensure the device defaults to a landscape experience for the user:
*   **Window Manager (WMS)**: Modify `DisplayRotation.java` to set the default user rotation to `ROTATION_90`.
*   **Settings Provider**: Ensure `def_user_rotation` is initialized to 1 (Landscape).
*   **System UI & Apps**: Optimize Boot Animation, Launcher, and Setup Wizard to prefer landscape orientation by default.

## 6. Verification Plan
The fix was verified through three iterative phases:
1.  **Functional Verification**: Confirming ARCore "Saliency" and "Tracking" status is "Healthy".
2.  **Regression Testing**:
    *   Camera UI orientation in vertical/horizontal switches.
    *   Recent Apps (Overview) thumbnail rendering.
    *   Setup Wizard (SUW) layout under various rotation locks.
3.  **Third-party Compatibility**: Validating QR scanners and photo editors (e.g., Google Photos, Instagram).

## 8. Implementation Details (Code Reference)
This section provides the verbatim code changes (diffs) implemented to achieve the architectural alignment.

### 8.1 System Configuration
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

### 8.2 HAL Layer Alignment
#### Touch Input (Kernel Driver)
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

#### Sensor HAL (Registry Configuration)
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

#### Camera (Device Tree)
**Patch:** `vendor/qcom/proprietary/0001-UPDATE-BSP-Display-Rotation-Change-camera-sensor-ori.patch`
```dts
--- a/camera-devicetree/yupik-camera-sensor-idp.dtsi
+++ b/camera-devicetree/yupik-camera-sensor-idp.dtsi
-		sensor-position-roll = <0>;
+		sensor-position-roll = <90>;
```

### 8.3 Framework & UI Compensation
#### Boot Animation
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

#### Window Manager Service (DisplayRotation)
**Patch:** `frameworks/base/0002-Update-BSP-Display-Rotation-Fixed-display-flipping-i.patch`
```java
--- a/services/core/java/com/android/server/wm/DisplayRotation.java
+++ b/services/core/java/com/android/server/wm/DisplayRotation.java
+        // Customer offset: add +90° for internal default panel only
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
