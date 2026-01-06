# RCA 第四點：Hypothesis Loop（假設驗證迴圈）實戰教材

> 適用：Android BSP / Kernel / Driver / HAL / Power / CTS / Field Issue

---

# 🎯 目的

> 用 **最小成本、最快速度** 縮小問題範圍，直到只剩下一個最可能的原因。

在 RCA 流程中：

```
Repro + Evidence
        ↓
  Hypothesis Loop   ← 工程師的主戰場
        ↓
   Root Cause
```

Hypothesis Loop =

> 提出假設 → 設計實驗 → 驗證 → 排除 → 再提出更小範圍的假設 → 不斷收斂

---

# 🧠 什麼是「假設」？

假設不是亂猜，而是：

> **根據 log、trace、現象、系統架構，推導「可能出問題的區塊」**

常見例子：
- 這可能是 suspend/resume race
- 這可能是 PMIC brownout
- 這可能是 modem crash 觸發 AP reset
- 這可能是 framework service deadlock

---

# 🧩 4.1 問題分類套路（先分類再下手）

不同問題有固定切入點：

| 現象 | 第一優先檢查 |
|------|--------------|
| Reboot | ramoops / last_kmsg / watchdog |
| Crash | tombstone / callstack |
| 卡死 | ANR traces / binder |
| 耗電 | wakelock / suspend state |
| 充電 | PMIC / charger / power supply class |
| CTS fail | assertion / spec mismatch |

### 範例：Reboot

第一步一定是：

```bash
adb shell cat /sys/fs/pstore/console-ramoops
```

確認：
- kernel panic?
- watchdog?
- subsys restart?
- brownout?

---

# 🪓 4.2 最小變更法（Binary Search 思維）

> **一次只動一個變數**

## 常用手法

### ① Good build vs Bad build

- 上一版 OK，這一版壞 → 中間一定有 commit
- 使用：

```bash
git bisect
```

---

### ② Feature on/off

- 關閉：
  - 某個 driver
  - 某個 HAL
  - 某個 service
  - 某個 power feature

觀察：
> 問題是否消失或行為改變？

---

### ③ DTS / Config 切換

- 關 runtime PM
- 關 autosuspend
- 改 regulator mode
- 關 clock gating

---

### ④ 外部條件替換

- 換 charger
- 換 cable
- 換 SIM
- 換 module
- 換 board

---

# 🧨 4.3 每個假設都必須「可被證偽」

> ❌ 錯誤：  
>「我覺得是 driver bug」

> ✅ 正確：  
>「如果是這個 driver 的 race，那 **關掉 autosuspend 或加鎖後，問題應該消失或行為改變**」

## 實例

假設：
> display driver suspend race

實驗：
```bash
echo on > /sys/devices/.../power/control
```

結果：
- 問題消失 → 命中
- 問題仍在 → 排除

---

# 🧱 4.4 完整實戰案例：插拔充電器 reboot

## 現象
- 插拔 charger 會 reboot

## 第一層假設
- A：kernel panic
- B：PMIC brownout
- C：modem crash

### 驗證

| 假設 | 驗證方式 | 結果 |
|------|-----------|--------|
| A | 看 ramoops | 有 panic |
| B | 看 PMIC log | 無 |
| C | 看 modem SSR | 無 |

→ 鎖定 kernel

---

## 第二層假設

- A1：charger driver
- A2：power supply framework
- A3：thermal

### 實驗
- Disable charger IRQ

→ 問題消失

---

## 第三層假設

- IRQ handler vs suspend race

→ 加 lock + trace → 確認命中

---

# 🧠 工程心法

> 你不是在「找答案」，你是在「系統性排除不可能」。

---

# 📌 何時可以結束 Hypothesis Loop？

當你可以說出：

> 「**如果沒有這個錯誤，這個問題就不會發生**」

而且：
- 移除這段 code → 問題消失
- 修正這個條件 → 問題消失

---

# ❌ 常見失敗模式

- 同時改多個變數
- 沒有設計可證偽實驗
- 一直換 log，沒有假設
- 用猜的，不用實驗

---

# ✅ 一句話總結

> **Hypothesis Loop = 用實驗逼系統說實話**

---