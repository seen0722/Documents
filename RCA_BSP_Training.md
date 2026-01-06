# ODM BSP 團隊 RCA（Root Cause Analysis）標準流程教材

> 適用範圍：Android BSP / SoC Platform / Kernel / HAL / Driver / System / Factory / GMS / CTS / Field Issue

---

# 🎯 目標

建立一套：
- 可重複
- 可訓練新人
- 可跨部門協作
- 可對客戶與 IC vendor 交付

的 **標準 RCA 分析流程**

---

# 📦 RCA 的輸入與輸出

## 輸入（Trigger）
- EVT / DVT / PVT fail
- CTS / VTS / GTS fail
- Customer issue / Field return
- Random reboot / Crash / ANR / Power / Thermal / Charging / Camera / WWAN / Sensor

## 輸出（Deliverables）
- Root Cause（根因）
- Fix Plan（短期 + 長期）
- Verification Plan（驗證方式）
- Prevention（預防再發生）

---

# 🧭 RCA 總流程

```
Issue 
  → Triage 
    → Repro 
      → Evidence 
        → Hypothesis 
          → Root Cause 
            → Fix 
              → Verify 
                → Prevent
```

---

# 1️⃣ Triage（問題定義與分流）

## 1.1 問題敘述模板（Problem Statement）

> 在【什麼環境 / 條件】下  
> 發生【什麼行為】  
> 導致【什麼結果 / 影響】

範例：
```
在 DVT build + 低電量 + 特定 charger 情境下，
插拔充電器會造成系統 reboot，
導致裝置無法正常充電。
```

---

## 1.2 嚴重度分級（Severity）

| Level | 定義 |
|------|------|
| S0 | 無法開機 / 安規 / 法規 / 量產阻斷 |
| S1 | 核心功能不可用 / CTS 阻斷 / 頻繁 crash |
| S2 | 次要功能異常 / 有 workaround |
| S3 | UI / 體驗 / 輕微問題 |

---

## 1.3 Ownership（責任歸屬）

- Kernel
- Vendor HAL
- Framework
- App
- Hardware
- Factory
- 3rd party module

---

# 2️⃣ Repro（重現問題）

> **RCA 成敗 80% 取決於是否能穩定 Repro**

## 2.1 建立 Golden Repro

必須記錄：
- Build fingerprint
- 機型 / Board rev
- 外設版本
- 操作步驟
- 環境條件（溫度 / 電量 / 網路）

目標：**第三人可在 30 分鐘內重現**

---

## 2.2 Repro Rate

- 10/10
- 3/10
- 1/50

偶發問題建議：
- Loop test
- Stress test
- Monkey
- Soak test
- Suspend/Resume loop
- Plug/unplug loop

---

## 2.3 無法 Repro 時

- 轉 Evidence-based RCA
- 補 log / trace / debug flag
- 先讓問題「可觀測」

---

# 3️⃣ Evidence Package（證據包）

## 3.1 必備

- logcat -b all
- dmesg / kernel log
- ramoops / last_kmsg
- tombstone / ANR traces
- bugreport
- build version / commit / manifest

---

## 3.2 進階（依問題類型）

- perfetto / systrace
- ftrace / trace-cmd
- power / charger IC log
- RF log
- memory dump / kmemleak
- I2C / SPI analyzer

---

# 4️⃣ Hypothesis Loop（假設驗證迴圈）

## 4.1 問題分類套路

| 類型 | 優先檢查 |
|------|----------|
| Reboot | panic / watchdog / brownout |
| Crash | tombstone / callstack |
| Power | regulator / suspend / resume |
| Driver | I2C timeout / IRQ / race |
| CTS | assertion / spec mismatch |

---

## 4.2 最小變更法（Binary Search）

- good build vs bad build
- feature on/off
- config / DTS 切換
- 換模組 / 換 charger / 換 cable

---

## 4.3 每個假設都必須能被「證偽」

---

# 5️⃣ Root Cause 表達方式

## 5.1 5 Whys

```
為什麼 reboot？
→ 因為 kernel panic

為什麼 panic？
→ 因為 NULL pointer dereference

為什麼 NULL？
→ 因為 suspend race condition

為什麼有 race？
→ 因為 state machine 沒有 lock

為什麼沒 lock？
→ 因為缺乏 power concurrency 設計規範
```

---

## 5.2 Causal Chain（因果鏈）

- 觸發條件
- 必要條件
- 放大條件
- 防護機制失效點

---

# 6️⃣ 修復策略

## 6.1 修復分級

- Workaround（快速止血）
- Tactical Fix（修 bug）
- Strategic Fix（架構改善）

---

## 6.2 風險評估

- 是否動到 common path
- 是否影響 CTS / ABI / KMI
- 是否容易回滾

---

# 7️⃣ 驗證與回歸

## 7.1 Verification

- Repro case 至少 pass N 次
- 關鍵 log / trace 符合預期

## 7.2 Regression

- Sanity
- Stress
- Soak
- CTS / VTS subset
- Factory test

---

# 8️⃣ RCA 報告模板

## 必填章節

- Summary
- Impact
- Repro Steps
- Evidence
- Root Cause
- Fix
- Verification
- Regression
- Prevention

---

# 9️⃣ Prevention（避免再犯）

## 常見改善項目

- Code review checklist
- Power / concurrency test
- Nightly stress
- 可觀測性指標
- CTS gate
- Factory 防呆測項

---

# 🏁 結語

> 好的 RCA 不是「找到誰寫錯」，  
> 而是「讓同樣的錯誤不再發生」。

---

# 📎 建議工具

- perfetto
- trace-cmd
- ramoops
- bugreport
- CTS / VTS
- git bisect
- stress-ng / monkey

---