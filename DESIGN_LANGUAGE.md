# 設計語言指南 — Yang Yang 的 App 風格底層邏輯

這份文件整理自 ProHabs App 的實際程式碼，記錄開發者一貫的設計決策與互動邏輯。
開發新 App 前，請先讀完這份文件，讓 AI 或合作者對齊風格，不需從頭溝通。

---

## 1. 顏色系統

### 主色調
```
--bg:            #FFFFFF        白底，乾淨
--card:          #F4FAF2        極淡綠，卡片底色
--primary:       #2EAD5C        主綠，品牌色
--primary-light: #C8F0D8        淡綠，完成狀態底色 / badge 背景
--text:          #1A3319        深綠黑，主文字
--text-sub:      #7A9070        中灰綠，輔助文字 / 標籤
--shadow:        0 2px 12px rgba(46,173,92,0.10), 0 1px 3px rgba(0,0,0,0.05)
```

### 語意色彩
- **金幣 / 獎勵**：`#FFC200`（黃）, `#92680A`（深褐文字）, `#FFFBE8`（淡黃底）
- **危險 / 刪除**：`#E53935`（紅）, `#FDECEA`（淡紅底）
- **暫停 / 警告**：`#F59E0B`（琥珀）
- **配對習慣**：`rgba(255,194,0,0.15)` 底 + `#92680A` 文字

### 習慣顏色主題（8色）
每個習慣有獨立顏色，包含 4 個 token：
```
bg（卡片底色）/ done（完成底色）/ border（邊框）/ accent（打勾圓、文字強調）
```
透過 CSS 自定義變數注入元素：`--hc-bg`, `--hc-done`, `--hc-border`, `--hc-accent`
所有習慣相關元件都使用這四個變數，而不是寫死顏色，讓顏色主題完整穿透。

```
red / orange / yellow / green / teal / blue / purple / pink
```

---

## 2. 圓角哲學

圓角大小代表「層級距離」，越大越近使用者：

| 場景 | 圓角值 |
|------|--------|
| 全螢幕底部 sheet 上緣 | `28px` |
| 中央彈窗 | `24px` |
| 主卡片（card、stat 卡） | `20px`（`--radius-card`） |
| 習慣卡片 | `16px` |
| 按鈕、輸入框 | `14px`（`--radius-btn`） |
| 小 badge / chip / tag | `99px`（膠囊形，全圓） |
| 圓形元素（check circle、FAB） | `50%` |

---

## 3. 動畫原則

### 彈性 spring（用於「有重量感」的動作）
```css
cubic-bezier(0.34, 1.56, 0.64, 1)   /* 彈出感，用於 modal 出現、check 勾選 */
cubic-bezier(0.34, 1.2, 0.64, 1)    /* 輕彈，用於 bottom sheet 上滑 */
cubic-bezier(0.34, 1.1, 0.64, 1)    /* 微彈，用於對話框縮放 */
```

### 常用時長
- overlay fade in/out：`0.25s`
- modal slide up：`0.3s`
- 卡片按壓縮放：`0.15s`
- check circle 勾選：`0.2s`
- 進度條寬度：`0.4–0.5s ease`
- 金幣 rewardPop：`0.35s` + scale(0.7→1) spring
- 金幣圖片 spin：`0.6s` rotateY 360°

### 按壓回饋
- 所有可點擊卡片按下時：`transform: scale(0.985)` + 陰影變小
- 按鈕 active：`opacity: 0.85`
- FAB active：`scale(0.92)`

### Ripple 效果
點擊習慣卡片時，產生從手指位置擴散的半透明圓形波紋（`scale: 0→4`, `opacity→0`, 0.5s）。

---

## 4. 版面結構（Mobile-first PWA）

- 最大寬度：`430px`，水平置中
- 底部導航：固定浮動，距底部 `12px + env(safe-area-inset-bottom)`，圓角膠囊形（`border-radius: 22px`），非全寬
- 頁面 padding-bottom：`--nav-h (80px) + safe-area-inset-bottom`，避免內容被導航遮住
- Header padding-top：`max(env(safe-area-inset-top), 20px)`，iPhone 瀏海安全
- 捲動區塊隱藏捲軸：`::-webkit-scrollbar { display: none }`

---

## 5. Modal / Sheet 模式

### Bottom Sheet（從底部滑上）
- 預設狀態：`opacity: 0; pointer-events: none; translateY(40px)`
- 開啟狀態：`opacity: 1; translateY(0)`
- 上方有「handle 條」：`width: 36px; height: 4px; background: #DDD; border-radius: 2px`
- 支援**下滑關閉**：偵測 touchstart → touchend，若 `dy > 80px` 且 scroll 在頂部則關閉

### Center Dialog（中央縮放出現）
- 預設：`transform: scale(0.9); opacity: 0`
- 開啟：`transform: scale(1)`
- 用於**確認類操作**（購買確認、刪除確認）

### 通用規則
- overlay 背景：`rgba(0,0,0,0.35~0.5)`
- 關閉方式：點擊 overlay 外部 或 明確取消按鈕，**不支援 Escape 鍵**（行動優先）

---

## 6. 拖曳排序系統

### Ghost 元素
拖曳時，原位置項目變淡（opacity 0.4, scale 0.98），同時產生跟隨手指的浮動 ghost：
```css
position: fixed; pointer-events: none; z-index: 999;
background: white; border-radius: 20px;
box-shadow: 0 8px 32px rgba(0,0,0,0.18);
transform: rotate(1.5deg) scale(1.02);   /* 輕微歪斜，感覺「被拿起來」 */
```

### Drop Indicator
插入位置顯示一條線（非虛線框），設計如下：
```css
height: 3px; background: var(--primary); border-radius: 99px;
margin: -2px 4px;
/* 左端有圓點 */
::before { width: 11px; height: 11px; border-radius: 50%; background: var(--primary); left: -4px; top: -4px; }
```
這條線出現在**準確的插入位置**，根據手指相對於目標元素上/下半部判斷。

### 觸發方式
- 習慣排序：長按拖曳 handle（六點圖示）
- 鏈條排序：長按鏈條 header 右側的拖曳 handle
- 獨立習慣可拖入鏈條（手指移進鏈條內部區域時，indicator 出現在鏈條內）
- 鏈條內習慣可拖出成為獨立習慣

---

## 7. 刪除 / 危險操作的安全機制

任何不可逆操作**必須二次確認**，不可直接執行。確認方式分兩種：

### Modal 確認（高風險）
用於：刪除習慣、全部清除紀錄
- 開啟一個 center dialog
- 說明後果（紅色粗體「無法復原」）
- 確認按鈕為紅色（`#E53935`），取消在左
- 示例文字：`「確定要刪除「XXX」嗎？所有打卡紀錄也會一併刪除。」`

### Native confirm()（低風險）
用於：刪除 tag、匯入備份覆蓋
- 使用瀏覽器原生 `confirm()` dialog
- 說明覆蓋影響範圍

### 規則：刪除鏈條不刪習慣
刪除鏈條時，其中的習慣**自動變為獨立習慣**，不連帶刪除。這保護用戶資料。

---

## 8. 可點擊元素的視覺語言

### 「可以點這裡設定」的暗示
管理頁中習慣名稱使用虛線底線（`.tappable-name`）暗示可點擊進入設定：
```css
border-bottom: 1.5px dashed #B0D4B8; cursor: pointer;
```

### 「可以新增」的暗示
空白新增區域使用虛線邊框（不是實線）表示「這裡可以加東西」：
```css
border: 2px dashed rgba(46,173,92,0.3); background: rgba(46,173,92,0.08);
```

### 標籤 / Badge
- 功能 badge（清單、鏈條名稱）：`border-radius: 99px`, 淡色背景，字體 10–11px, font-weight 700
- 狀態 badge（今日待完成、已完成）：同上，顏色語意化（黃=待辦, 綠=完成）

---

## 9. 長按手勢

長按用於**進入特殊模式**，而不是直接觸發動作：

- 習慣卡片長按（500ms）→ 進入「配對模式」，顯示固定在頂部的提示條
- 提示條：固定定位、膠囊形、黃色背景、高 z-index，說明下一步操作
- 配對第二個習慣後，自動退出模式並顯示 Toast 確認

長按觸發後，同一個 touch 序列的後續 click 事件需**忽略**（防止誤觸）。

---

## 10. Toast 通知

兩種 Toast，用途不同：

### 一般 Toast（操作回饋）
```css
position: fixed; bottom: calc(nav-h + 16px);
background: #2C1A0E; color: white; border-radius: 99px;
opacity 0→1 動畫，2.5 秒後消失
```
用途：配對成功、解除配對、備份完成、清除完成

### 獎勵 Toast（金幣通知）
```css
position: fixed; top: 80px;  /* 從上方滑入 */
background: #1A1A1A; border-radius: 99px;
translateY(-20px)→0 動畫
金額文字：#FFC200 黃色, font-weight 800
副標文字：rgba(255,255,255,0.7)
```
用途：打卡獲得金幣、計時結束獲得金幣

---

## 11. 鏈條視覺設計

今日頁中，鏈條內的習慣透過以下元素連結：

- **左側豎線**：`position: absolute; left: 0; width: 3px; background: var(--primary); opacity: 0.3; top: 36px; bottom: 10px`（不從頂部起，留出 header 高度）
- **項目間小箭頭**：`opacity: 0.5; color: var(--primary)`，視覺上連結上下兩個習慣
- **鏈條完成橫幅**：全綠底色 banner，`animation: popIn 0.4s spring`

---

## 12. 排版規則

- 主字體：`Noto Sans TC`（繁體中文優先）
- 頁面大標題：26px, font-weight 700, letter-spacing -0.8px
- 卡片習慣名稱：15px, font-weight 500
- 輔助標籤（section label）：11–12px, font-weight 700, letter-spacing 0.5–0.8px, text-transform uppercase
- 選填說明文字：相同標籤內用 font-weight 400 + text-transform none 降調
- 數字（金幣餘額、連續天數）：font-size 30–48px, font-weight 800, letter-spacing -2px

---

## 13. 輸入框風格

```css
border: 2px solid transparent;   /* 預設無邊框感 */
background: var(--card);         /* 與頁面融合 */
border-radius: var(--radius-btn); /* 14px */
transition: border-color 0.2s;
:focus { border-color: var(--primary); }  /* 聚焦才顯示邊框 */
```
輸入框使用淡綠卡片底色，平時幾乎沒有邊框，聚焦後才出現品牌色邊框。這讓介面保持乾淨，不會因為一堆 input 邊框而視覺雜亂。

---

## 14. 狀態持久化

- **本地優先**：資料存 localStorage，鍵名帶版本號（如 `habitData_v2`）
- **雲端同步**：每次 `saveData()` 自動觸發 Supabase 推送（`pushToCloud()`），非同步，不阻塞 UI
- **資料向前相容**：`loadData()` 針對每個新欄位做 null check 補預設值，舊資料不會壞掉

---

## 15. 整體設計哲學摘要

1. **行動優先，非響應式**：設計目標是 iPhone，max-width 430px，不做桌面適配
2. **圓潤勝於稜角**：所有元素都有圓角，越重要的元素圓角越大
3. **彈性動畫**：使用 spring cubic-bezier，讓元素「有重量感地彈出來」
4. **危險操作雙重確認**：任何刪除/覆蓋操作都需要二次確認，且說清楚後果
5. **drag 有 ghost + indicator**：拖曳永遠顯示「被拿起的幻影」和「放下後的位置線」，用戶不用猜
6. **長按 = 進入模式**：長按是進入特殊互動模式的入口（配對、拖曳），不直接執行動作
7. **顏色主題穿透**：習慣顏色透過 CSS 變數貫穿整個卡片系統，不用為每個顏色寫重複的 class
8. **Toast 在地化**：Toast 出現位置有語意——操作回饋在底部，獎勵通知在頂部（用戶視線自然往上看）
9. **空狀態引導**：空鏈條不只顯示空白，而是有提示文字說明下一步
10. **刪鏈條保習慣**：結構改變時保護用戶資料，鏈條消失但習慣留下變獨立
