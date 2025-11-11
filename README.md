*電表階層管理系統 (PowerSystem)*

以 Vue 3 + Vite 建構的「電表階層管理系統」，提供電表（Meter）之階層結構管理、節點 CRUD、拖拉調整父子關係與規則驗證，協助在前端可視化地管理樹狀電表資料。

Quick start：npm install → npm run dev（更多見下方「專案啟動」）。

*特色功能*

樹狀電表管理：以樹狀介面呈現電表階層，支援展開/收合、搜尋/篩選。

節點 CRUD：新增、重新命名、刪除電表節點；可設定父子層級。

拖拉調整階層：以拖放（Drag & Drop）更換上層（re-parenting）。

嚴謹的移動規則：避免循環、避免無效移動（詳見下節）。

清楚的使用者回饋：違規動作會有明確訊息，並阻止提交。

前端就緒：以 Vite 開發伺服，秒開發、熱重載。

*核心規格*
資料模型

電表以樹狀表示，建議每個節點（MeterNode）至少包含：
type MeterId = string;

interface MeterNode {
  id: MeterId;          // 唯一識別
  name: string;         // 顯示名稱
  type?: 'main' | 'sub' | 'virtual'; // 類型（可依需求延伸）
  parentId: MeterId | null;          // 根節點為 null
  children: MeterNode[];             // 子節點
}
備註：若資料源自 API，可在前端以 parentId 生成樹；或直接由後端回傳已組裝之巢狀結構。

*拖拉/移動規則*

為了保護資料一致性，進行節點移動（A 拖到 B 底下）時，需通過以下規則：
不可將節點移動到「自身或其任一子孫節點」底下（避免循環階層）
不可移動到與原本相同的父層（避免無意義操作）

自訂規則（可選）：
相同類型是否能互相包含（例如 main 底下是否允許 main）
單一父節點的子數量上限
禁止跨專案/區域移動等商規

*錯誤處理與提示*

當偵測到違規情況時：
顯示清楚錯誤訊息（例如：「無法將節點移至自身或其子層底下」）。
禁止提交操作（不更新樹或還原拖拉）

*畫面示意*
<img width="1729" height="733" alt="電表1" src="https://github.com/user-attachments/assets/db18178e-407d-400b-91ac-d5b774961a61" />

<img width="1762" height="866" alt="電表2" src="https://github.com/user-attachments/assets/984b0c26-6565-42cb-b558-9aa02126ac34" />

*安裝與開發*
環境需求

Node.js ≥ 18（建議 LTS）
npm 或 pnpm / bun 其一

專案啟動
# 1) 安裝依賴
npm install

# 2) 啟動開發伺服（熱重載）
npm run dev

# 3) 生產建置
npm run build

*建置與部署*

產物輸出於 dist/
靜態主機（Netlify、Vercel、GitHub Pages）或任一支援靜態檔案的伺服器皆可部署
若未來接後端 API，請在 .env / .env.production 中配置 VITE_API_BASE_URL

*UI/UX 建議*

拖拉過程顯示允許/禁止的即時樣式
放手前即先行預檢（preview check），避免閃爍的回滾
提供鍵盤輔助操作（上下移動、縮排/反縮排改層級）

*測試建議*

單元測試：
isDescendant、getAncestors、moveNode、canMove
E2E 測試：
拖拉有效案例（A → B）、無效案例（A → A 子孫），訊息彈出、狀態不變
極端案例：
深層樹、上千節點、相同名稱不同 id

*Roadmap*

 節點多選、批次移動/刪除
 拖拉自動捲動、置入位置預視（插入於前/後/內）
 權限控管（唯讀/可編輯角色）
 後端整合（REST / GraphQL），支援樂觀更新與失敗回滾
 匯入/匯出（CSV/JSON）
 i18n（繁中/英文切換）

*授權*

此專案授權條款：MIT（或依你需求調整；若未指定，可先用 MIT 以利開源協作）

*備註*

目前 GitHub 頁面語言配比顯示以 Vue 為主，並含 JavaScript / HTML；啟動與建置指令來自 repo 頁面預設 README（Vue + Vite 模板）
