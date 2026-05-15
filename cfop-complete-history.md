# CFOP 除錯完整歷程 — 所有嘗試與死局
## 2026-05-15 最終版

---

## 一、問題描述

**魔方小老師 v3** 的 CFOP 訓練模式從未成功運行。每次掃描完成後選擇 CFOP 模式，都顯示「魔方小老師看走眼了！」。Kociemba 快速解完全正常。

---

## 二、已確認正常的模組

| 模組 | 驗證方式 | 結果 |
|------|---------|------|
| 相機掃描 + SCAN_CONFIG | 實際拍照測試 | ✅ |
| Worker validate（奇偶性驗證） | MCP 注入測試 | ✅ ok:true |
| Worker solve（Kociemba） | MCP 注入測試 | ✅ 22步 34ms |
| Worker autofix（暴力修正） | MCP 注入測試 | ✅ 找到 swap |
| syncFixedStateToCube | MCP 注入測試 | ✅ 正確回寫 |
| CFOP 引擎自測 | _lbl.dm 打亂→求解 | ✅ 5/5 通過 |
| **CFOP 引擎（掃描狀態）** | 掃描→注入→求解 | **❌ 16 格錯誤** |

---

## 三、根因定位

### 發現過程

用 MCP 對已解完的魔方施加 `_lbl.dm('U')`，比較結果：

```javascript
var s = _lbl.solved();
_lbl.dm(s, 'U');
console.log(s.F[0]); // 結果: 'B'（藍，來自 R 面）
```

用真實魔方驗證標準 U CW（從上方看順時針）：前面頂列應得到左面（綠色）。

| 操作 | F[0] 結果 | 來源面 |
|------|----------|--------|
| 標準 U CW | G（綠） | L 面 |
| `_lbl.dm('U')` | B（藍） | R 面 |

**R/L/F/B 四個面的 `_lbl.dm` 方向與標準一致，只有 U 和 D 是反的。**

### `_lbl.cyc` 原始碼

```javascript
// U: F←R, R←B, B←L, L←F （逆時針 — 與標準相反）
case'U':t=[s.F[0],s.F[1],s.F[2]];
  s.F[0]=s.R[0]; s.R[0]=s.B[0]; s.B[0]=s.L[0]; s.L[0]=t[0]; ...

// 標準 U CW 應為: F←L, L←B, B←R, R←F （順時針）
```

D 面同理。

---

## 四、嘗試過的修復方案與結果

### ❌ 方案 A：翻譯輸出步驟 (U↔U', D↔D')

**邏輯**：如果 `_lbl.dm('U')` = 標準 U'，那翻轉所有 U/D 步驟就能對齊。

**測試**：
```javascript
function translateMoves(arr) {
  return arr.map(m => {
    if(m.includes('2')) return m;
    if(m[0]==='U'||m[0]==='D')
      return m.includes("'") ? m.replace("'","") : m+"'";
    return m;
  });
}
```

**結果**：Kociemba 解法翻譯後施加 → **33 格錯誤**（比不翻譯的 2 格更糟）。

**死因**：`_lbl.dm('U')` 不等於標準 U 也不等於標準 U'。它是「CW 面旋轉 + CCW 鄰面循環」的混合操作，在物理世界中不存在。簡單翻譯的數學基礎不成立。

---

### ❌ 方案 B：State Desync（第三方建議）

**邏輯**：Auto-Fix 修正了 facelet 但沒同步回 faceColors。加入 `syncFixedStateToCube()` 就能解決。

**實作**：已加入 Worker autofix 命令 + syncFixedStateToCube 函數。

**測試**：用不需要 Auto-Fix 的合法狀態（validate ok:true）直接測 CFOP。

**結果**：仍然 **16 格錯誤**。

**死因**：問題不是同步，而是座標系差異。即使 faceColors 完全合法且同步，CFOP 引擎仍然無法解讀物理世界的狀態。

---

### ❌ 方案 C：直接修正 cyc(U) 和 cyc(D)（第三方建議）

**邏輯**：修正 cyc 使 U/D 方向與標準一致，公式會「自動跟著轉正」。

**實作**：用 MCP 即時注入修正後的 `_lbl.cyc`：
```javascript
// U: F←L（標準 CW）
case'U':...s.F[0]=s.L[0];s.L[0]=s.B[0];s.B[0]=s.R[0];s.R[0]=t[0];...
// D: F←R（標準 CW from below）
case'D':...s.F[6]=s.R[6];s.R[6]=s.B[6];s.B[6]=s.L[6];s.L[6]=t[0];...
```

**測試**：

| 測試 | 修正前 | 修正後 |
|------|--------|--------|
| `_lbl.dm('U')` 方向 | F←R ❌ | F←L ✅ |
| CFOP 自測（_lbl 打亂） | 0 diffs ✅ | **33 diffs ❌** |
| CFOP 掃描狀態 | 16 diffs ❌ | **41 diffs ❌** |

**死因**：修正 cyc 後**自測也壞了**。這證明 CFOP 引擎的所有邏輯（Cross 搜索、F2L 配對、OLL 簽章、PLL 匹配）都深度依賴舊的 U/D 行為。「公式自動轉正」的假設不成立，因為不只公式字串，整個辨識和搜索邏輯都需要改。

---

### ❌ 方案 D：輸入狀態座標轉換

**邏輯**：找到置換函數 P，使 `_lbl.cfopEngine(P(scanState))` 能正確求解。

**死因**：由於 `_lbl.dm('U')` 是物理上不存在的混合操作（CW面 + CCW鄰面），不存在簡單的位置重排能在兩個座標系之間轉換。

---

## 五、💡 新提案：Kociemba 橋接方案 (Bridge Pattern)

### 核心思路

既然 Kociemba 和 _lbl 雖然方向不同，但各自內部都是自洽的，就用 Kociemba 做翻譯橋：

```
物理世界的掃描狀態 S
  → Kociemba 求解得到步驟 M（標準座標系）
  → S 經過 M 可以還原為 solved
  → 反轉 M 得到 M⁻¹
  → 從 _lbl.solved() 出發，用 _lbl.dm 施加 M⁻¹
  → 得到 _lbl 宇宙中的「等價打亂狀態」S'
  → _lbl.cfopEngine(S') → 正常求解 ✅
```

### 為什麼行得通

1. Kociemba 的 M 在標準座標系下：`M(S) = I`（還原為恆等）
2. M⁻¹ 是 M 的逆序：`M⁻¹(I) = S`（從 solved 到達 S）
3. 用 `_lbl.dm` 施加 M⁻¹ 到 `_lbl.solved()`：在 _lbl 宇宙中產生一個打亂狀態 S'
4. S' 和 S 代表「相同的物理魔方」在不同座標系中的表達
5. `_lbl.cfopEngine(S')` 能正確求解，因為 S' 是由 `_lbl.dm` 產生的

### 虛擬碼

```javascript
function bridgeScanToCFOP(faceColors) {
  // 1. 產生 facelet
  var fl = '';
  ['U','R','F','D','L','B'].forEach(f =>
    faceColors[f].forEach(x => fl += C2F[x]));

  // 2. Kociemba 求解（透過 Worker）
  var solution = await workerSolve(fl);  // e.g. "U D2 F' ..."

  // 3. 反轉解法
  var moves = solution.split(' ');
  var reversed = moves.reverse().map(m => {
    if(m.includes("'")) return m.replace("'","");
    if(m.includes("2")) return m;
    return m + "'";
  });

  // 4. 從 _lbl.solved() 施加反轉解法
  var lblState = _lbl.solved();
  reversed.forEach(m => _lbl.dm(lblState, m));

  // 5. 這個 lblState 是 CFOP 引擎能理解的等價狀態
  return lblState;
}

// 使用：
var cfopState = await bridgeScanToCFOP(faceColors);
var result = _lbl.cfopEngine(cfopState);  // 應該能正確求解！
```

### 效能影響

| 步驟 | 耗時 |
|------|------|
| Worker solve | ~34ms（已初始化時） |
| 反轉 20 步 | <1ms |
| _lbl.dm × 20 | <1ms |
| **總額外成本** | **~35ms（使用者無感）** |

### 優點

- **零修改** _lbl 引擎核心 ✅
- **零修改** 57 OLL + 21 PLL 公式 ✅
- **零修改** OLL 簽章 HashMap ✅
- **零修改** Cross/F2L 搜索邏輯 ✅
- 只需加一個橋接函數 ✅

### 尚待驗證

> **方案 E 在群論上是否正確？**
>
> 假設 Kociemba 的標準操作集為 G_std = {U,U',D,D',R,R',L,L',F,F',B,B'}
> _lbl 的操作集為 G_lbl = {U_lbl, U'_lbl, D_lbl, D'_lbl, R,R',L,L',F,F',B,B'}
> 其中 R/L/F/B 在兩個集合中相同，但 U_lbl ≠ U_std 且 D_lbl ≠ D_std。
>
> 問題：如果 M = m1·m2·...·mk 是 G_std 中的解法（M(S) = I），
> 那麼 M⁻¹_lbl = _lbl.dm(mk⁻¹)·...·_lbl.dm(m1⁻¹) 施加到 _lbl.solved() 後，
> 得到的狀態 S' 是否能被 _lbl.cfopEngine 正確求解？
>
> 注意：M⁻¹_lbl 中的 U⁻¹ 步驟會被 _lbl.dm 以不同於標準的方式執行。
> 所以 S' ≠ S。但 S' 是 _lbl 宇宙中的合法打亂狀態，應該能被求解。

---

## 六、附件清單

| 檔案 | 版本 | 說明 |
|------|------|------|
| `integrated.html` | v3.5 | 當前整合版（Kociemba ✅，CFOP 掃描 ❌） |
| `cube-scan.html` | — | 獨立掃描器（調參用） |
| `cfop-debug-report.md` | v1 | 根因定位（U/D 方向差異） |
| `cfop-translation-deadend.md` | v2 | 翻譯方案失敗 |
| `cfop-final-proof.md` | v3 | State Desync 反證 |
| `cfop-debug-v4.md` | v4 | 修正 cyc 失敗 |
| `cfop-complete-history.md` | v5 | **本文件（完整歷程）** |

---

## 七、可重現的 MCP 測試指令

```javascript
// === 1. 驗證 _lbl.dm('U') 方向 ===
var s = _lbl.solved(); _lbl.dm(s, 'U');
console.log(s.F[0]); // 'B'=from R (反的), 'G'=from L (正的)

// === 2. 驗證 CFOP 自測 ===
var s2 = _lbl.solved();
['R','U','F','D','L','B',"R'","U'","F'","D'"].forEach(m=>_lbl.dm(s2,m));
var r = _lbl.cfopEngine(_lbl.clone(s2));
console.log(JSON.stringify(r.state) === JSON.stringify(_lbl.solved())); // true

// === 3. 驗證掃描狀態 CFOP 失敗 ===
faceColors = {U:['O','Y','O','R','W','R','R','B','O'],R:['Y','Y','W','Y','B','G','G','O','G'],F:['B','O','B','R','R','G','W','Y','O'],D:['R','B','Y','G','Y','W','R','W','W'],L:['Y','B','Y','G','G','W','B','W','G'],B:['B','O','G','O','O','R','R','B','W']};
var r2 = _lbl.cfopEngine(_lbl.clone(faceColors));
var diffs=0;var sv=_lbl.solved();
['U','R','F','D','L','B'].forEach(f=>{for(var i=0;i<9;i++)if(r2.state[f][i]!==sv[f][i])diffs++;});
console.log('diffs:', diffs); // 16

// === 4. 驗證修正 cyc 後自測壞掉 ===
// (hot-patch _lbl.cyc with F←L for U, F←R for D)
// 重跑自測 → diffs: 33

// === 5. 驗證 Kociemba 橋接方案（待測） ===
// workerSolve(fl) → reverse moves → _lbl.dms(solved, reversed)
// → cfopEngine → check solved
```
