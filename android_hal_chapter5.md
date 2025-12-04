# Part A · Chapter 5 --- 如何撰寫 HAL（AIDL + HIDL 全實作）

------------------------------------------------------------------------

## 5.1 HAL 實作流程總覽

所有 HAL（不論 AIDL/HIDL）都包含：

    I. Interface 定義（.aidl / .hal）
    II. HAL Service（C++ 實作）
    III. Service 註冊流程（binder / hwbinder）
    IV. init.rc 啟動 script
    V. VINTF manifest 宣告
    VI. Android.bp 建置
    VII. 測試與 debug

------------------------------------------------------------------------

# 🟩 5.2 AIDL HAL 實作（完整範例）

以下示範建立：

    android.hardware.demo.IDemo/default

------------------------------------------------------------------------

## 5.2.1 Step 1：撰寫 AIDL 介面

`hardware/interfaces/demo/aidl/IDemo.aidl`

``` aidl
package android.hardware.demo;

@VintfStability
@version 1
interface IDemo {
    String getMessage();
    int add(int a, int b);
}
```

------------------------------------------------------------------------

## 5.2.2 Step 2：Android.bp 產生 AIDL C++ 程式碼

``` bp
aidl_interface {
    name: "android.hardware.demo",
    srcs: ["IDemo.aidl"],
    backend: {
        cpp: { enabled: true },
    },
    versions: ["1"],
}
```

AIDL 自動產生： - IInterface\
- BnDemo\
- BpDemo\
- Parcelable types

------------------------------------------------------------------------

## 5.2.3 Step 3：HAL Service 實作（C++）

`Demo.cpp`

``` cpp
#include "IDemo.h"
using ::aidl::android::hardware::demo::IDemo;
using ndk::ScopedAStatus;

class Demo : public IDemo {
public:
    ScopedAStatus getMessage(std::string* _aidl_return) override {
        *_aidl_return = "Hello from Demo HAL!";
        return ScopedAStatus::ok();
    }

    ScopedAStatus add(int32_t a, int32_t b, int32_t* _aidl_return) override {
        *_aidl_return = a + b;
        return ScopedAStatus::ok();
    }
};
```

------------------------------------------------------------------------

## 5.2.4 Step 4：啟動 HAL Service（main）

`demo_service.cpp`

``` cpp
#include "Demo.cpp"
#include <android/binder_manager.h>
#include <android/binder_process.h>

using ::aidl::android::hardware::demo::IDemo;

int main() {
    ABinderProcess_setThreadPoolMaxThreadCount(4);

    std::shared_ptr<IDemo> demo = ndk::SharedRefBase::make<Demo>();
    const std::string instance =
        std::string() + IDemo::descriptor + "/default";

    AServiceManager_addService(
        demo->asBinder().get(),
        instance.c_str()
    );

    ABinderProcess_joinThreadPool();
    return 0;
}
```

------------------------------------------------------------------------

## 5.2.5 Step 5：init.rc（啟動 service）

`/vendor/etc/init/android.hardware.demo-service.rc`

``` rc
service vendor.demo-hal /vendor/bin/hw/android.hardware.demo-service
    class hal
    user root
    group root
    seclabel u:r:hal_demo_default:s0
```

------------------------------------------------------------------------

## 5.2.6 Step 6：VINTF manifest 宣告

`/vendor/etc/vintf/manifest.xml`

``` xml
<hal format="aidl">
    <name>android.hardware.demo</name>
    <version>1</version>
    <interface>
        <name>IDemo</name>
        <instance>default</instance>
    </interface>
</hal>
```

------------------------------------------------------------------------

## 5.2.7 Step 7：驗證 HAL 啟動

    adb shell service list | grep demo

結果：

    android.hardware.demo.IDemo/default

呼叫：

    adb shell dumpsys android.hardware.demo.IDemo/default

------------------------------------------------------------------------

# 🟦 5.3 HIDL HAL 實作（完整範例）

------------------------------------------------------------------------

## 5.3.1 Step 1：撰寫 HIDL 介面

`hardware/interfaces/demo/1.0/IDemo.hal`

``` hal
package android.hardware.demo@1.0;

interface IDemo {
    getMessage() generates (string msg);
    add(int32_t a, int32_t b) generates (int32_t result);
};
```

------------------------------------------------------------------------

## 5.3.2 Step 2：使用 hidl-gen 產生程式碼

    hidl-gen -Lc++ -randroid.hardware:hardware/interfaces android.hardware.demo@1.0

輸出： - BpHwDemo\
- BnHwDemo\
- IDemo.h

------------------------------------------------------------------------

## 5.3.3 Step 3：實作 HAL

``` cpp
#include <android/hardware/demo/1.0/IDemo.h>

using namespace android::hardware::demo::V1_0;

struct Demo : public IDemo {
    Return<void> getMessage(getMessage_cb _hidl_cb) override {
        _hidl_cb("Hello from HIDL Demo HAL!");
        return Void();
    }

    Return<int32_t> add(int32_t a, int32_t b) override {
        return a + b;
    }
};
```

------------------------------------------------------------------------

## 5.3.4 Step 4：啟動 HAL（main）

Binderized：

``` cpp
int main() {
    configureRpcThreadpool(1, true);
    sp<IDemo> service = new Demo();
    service->registerAsService();
    joinRpcThreadpool();
}
```

------------------------------------------------------------------------

## 5.3.5 Step 5：init.rc

``` rc
service vendor.demo-hal /vendor/bin/hw/android.hardware.demo@1.0-service
    class hal
```

------------------------------------------------------------------------

## 5.3.6 Step 6：VINTF 宣告

``` xml
<hal format="hidl">
    <name>android.hardware.demo</name>
    <version>1.0</version>
    <interface>
        <name>IDemo</name>
        <instance>default</instance>
    </interface>
</hal>
```

------------------------------------------------------------------------

## 5.3.7 Step 7：檢查 HAL 是否啟動成功

    adb shell lshal | grep demo

結果：

    android.hardware.demo@1.0::IDemo/default

------------------------------------------------------------------------

# ✔ Chapter 5 小結

### 🔹 AIDL HAL 特點

-   使用 Binder IPC
-   InterfaceVersion + Hash
-   API 可新增不可刪除
-   CTS/VTS 完整支援

### 🔹 HIDL HAL 特點

-   使用 HwBinder
-   介面版本用 @1.0 / @1.1 / @2.0
-   常用於 Camera、Audio、DRM 等舊 HAL

### 🔹 HAL 實作固定流程

1.  撰寫 interface\
2.  撰寫 C++ server class\
3.  註冊到 service manager\
4.  init.rc 啟動\
5.  VINTF 宣告\
6.  建置 & 測試
