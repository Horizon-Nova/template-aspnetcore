# ASP.NET Core MVC 架構規範

| 項目 | 內容 |
| --- | --- |
| 文件編號 | MVC-ARCH-001 |
| 版本 | 1.0 |
| 適用範圍 | ASP.NET Core MVC |
| 規範強度 | 強制，無例外 |
| 目標讀者 | 開發者、AI 代理人 |

---

## §1 規範用語

| 用語 | 意義 |
| --- | --- |
| **必須 (MUST)** | 強制條款，違反即為不合規 |
| **禁止 (MUST NOT)** | 強制禁止，無任何例外 |

---

## §2 專案目錄結構

以下是本專案唯一正確的目錄結構。AI 代理人不得新增層次、不得在禁止子資料夾的層內建立子資料夾。

```
Net/YourProject/
├── Areas/
│   └── YourWeb/
│       ├── Controllers/
│       ├── Services/
│       ├── Repositories/
│       ├── Core/
│       ├── Utilities/
│       └── Views/
│           └── Test/
│               ├── Index.cshtml        ← 必須
│               └── Partials/
│                   ├── _Styles.cshtml  ← 必須
│                   └── _Scripts.cshtml ← 必須
├── Extensions/
├── Middleware/
├── Filters/
├── Models/
├── wwwroot/
└── Program.cs
```

## §3 檔案命名規則

### §3.1 各層強制後綴

每個功能主題（如 Settings、License、Workspace）在每一層只能有一個對應檔案，以固定後綴命名。

**必須 (MUST)：**同一主題在同一層只能有一個檔案，且檔名後綴必須完全符合下表（以主題 `Settings` 為例）。

| 層次 | 強制後綴 | 正確 | 禁止 |
| --- | --- | --- | --- |
| `Middleware` | `Middleware` | `SettingsMiddleware.cs` | `SettingsRequestMiddleware.cs` |
| `Filters` | `Filters`（複數） | `SettingsFilters.cs` | `SettingsFilter.cs` |
| `Services` | `Services`（複數） | `SettingsServices.cs` | `SettingsPageService.cs`、`SettingsRuntimeService.cs` |
| `Repositories` | `Repositories`（複數） | `SettingsRepositories.cs` | `SettingsRepository.cs`、`SettingsReadRepository.cs` |
| `Models` | `Model` | `SettingsModel.cs` | `AppSettings.cs`（置於 Models 層時） |
| `Utilities` | `Utilities`（複數） | `SettingsUtilities.cs` | `SettingsHelper.cs`、`SettingsManager.cs` |
| `Core` | `Core` | `SettingsCore.cs` | `SettingsKernel.cs`、`SettingsCommon.cs` |

**禁止 (MUST NOT)：**
- 以子資料夾規避「一個主題，一個檔案」限制（例如 `Utilities/Settings/`、`Models/Settings/`）。

### §3.2 一個主題，一個檔案

同一功能主題在同一層只能有一個檔案。若同主題存在多個類別，合併至同一檔案。

```
❌ Services/SettingsPageService.cs + Services/SettingsRuntimeService.cs
✅ Services/SettingsServices.cs（合併）

❌ Model/Settings/AppSettings.cs（子資料夾）
✅ Model/SettingsModel.cs（扁平，一個檔案可含多個相關 record/class）

❌ Utilities/Settings/SettingsPolicy.cs（錯誤後綴 + 子資料夾）
✅ Utilities/SettingsUtilities.cs
```

### §3.3 函式命名前綴

函式名稱必須透過前綴明確表達動作性質，讀者不進入函式體即可判斷行為。

| 層次 | 允許前綴 | 語意 | 範例 |
| --- | --- | --- | --- |
| 前端（Razor Script / JavaScript） | `Search`、`Submit`、`Delete`、`bind`、`apply`、`toggle`、`show`、`close` | 畫面互動、事件綁定、Ajax 觸發、UI 狀態切換 | `Search()`、`Submit()`、`bindRoleFilters()` |
| Controllers | `Index`、`Search`、`Load`、`Submit`、`Delete`、`Refresh`、`Open` | HTTP 端點入口、頁面回傳、Partial 載入、提交處理 | `Index()`、`SearchItems()`、`LoadEditForm()` |
| Services | `Load`、`Create`、`Update`、`Delete`、`Save`、`Apply`、`ViewBag` | 組合呼叫、提供穩定入口，不承載業務規則 | `LoadItemList()`、`CreateItem()`、`ViewBagModel()` |
| Utilities | `Validate`、`Calculate`、`Build`、`Convert`、`Format`、`Resolve`、`Check` | 重邏輯、規則判斷、資料轉換、計算 | `CalculateScore()`、`ResolveScope()` |
| Repositories | `Query`、`Insert`、`Update`、`Delete`、`Exists` | 資料庫查詢與持久化操作 | `QueryItemList()`、`InsertItem()` |

**必須 (MUST)：**
- 函式名稱以動詞開頭，且動詞需能對應回傳值與副作用。
- 同一層必須優先使用該層既定前綴，不得任意混用其他層前綴。
- 非同步方法以 `Async` 結尾（例如 `QuerySettingsAsync()`、`LoadUserListAsync()`）。

**禁止 (MUST NOT)：**
- 使用 `Process`、`Handle`、`Do`、`Execute`、`Run`、`Manager` 作為函式名稱主動詞。
- 在 `Repositories` 使用 `Load`，或在 `Services` 使用 `Query`，造成責任語意混淆。

---

## §4 各層職責

### §4.1 依賴方向

依賴方向必須由外向內，外層可依賴內層，內層不得依賴外層；不得形成循環依賴。

```
Middleware / Filters
        ↓
Controllers → Services → Repositories → Models
                    ↓
               Utilities / Core
```

- 禁止跳層：`Views` / `Controllers` 不得直接呼叫 `Repositories`
- 禁止跳層：`Views` 不得呼叫 `Services` / `Repositories` / `Utilities` / `Core`
- 禁止反向：`Repositories` 不得依賴 `Services` / `Controllers`
- 禁止反向：`Utilities` / `Core` 不得依賴 `Middleware` / `Filters` / `Controllers` / `Services` / `Repositories`
- 禁止反向：`Models` 不得依賴任何其他層（若需共用型別，必須抽至 `Core`）

**必須 (MUST)：**允許的依賴方向如下（由外到內）：

- `Middleware` → `Filters` / `Services` / `Core` / `Utilities`
- `Filters` → `Services` / `Core` / `Utilities`
- `Controllers` → `Services` / `Core` / `Utilities`
- `Services` → `Repositories` / `Core` / `Utilities`
- `Repositories` → `Models` / `Core` / `Utilities`
- `Models` →（原則上不依賴其他層；如需共用型別，只能依賴 `Core` 的純型別）
- `Utilities` / `Core` →（不得依賴 `Middleware` / `Filters` / `Services` / `Repositories` / `Models` 之任何具體實作）

**禁止 (MUST NOT)：**
- `Repositories` 依賴 `Services`（禁止於 Repository 內呼叫 Service）。
- `Utilities` / `Core` 依賴任何 MVC/Web/HTTP 具體型別（避免核心能力被框架綁死）。
- `Models` 直接進行資料庫存取（資料庫互動必須集中於 `Repositories`）。

### §4.2 Middleware

**職責**：HTTP 請求進入 MVC 管線前後的橫切處理（請求/回應攔截、Header/Trace、例外攔截、授權前置檢查、請求記錄）。

**允許**：
- 讀取/寫入 `HttpContext`（例如 `Items`、`Headers`、`User`、`Response`）
- 呼叫 `Services`（僅用於「操作入口」型工作；不得在 Middleware 內實作業務判斷）

**禁止**：
- 直接存取資料庫（不得使用 `DbContext` / 任何 Repository 的具體資料存取）
- 實作業務規則判斷（判斷與計算必須下推至 `Utilities`）

### §4.3 Filters

**職責**：MVC 框架層級的橫切控制（授權/資源/動作/結果/例外），用來處理「與 MVC 執行流程強相關」的事項。

**允許**：
- 在 Filter 中做輕量的請求/回應處理（例如審計欄位補齊、統一錯誤回應格式）
- 呼叫 `Services` 取得必要資料（不得直接進行業務判斷）

**禁止**：
- 在 Filter 內做複雜邏輯與規則（必須下推至 `Utilities`）
- 在 Filter 內直接存取資料庫

### §4.4 Controllers

**職責**：HTTP 端點的薄層入口，負責路由、輸入模型繫結、授權判定結果處理、選擇回傳格式（View/JSON/Redirect）。

**必須 (MUST)：**
- 透過建構式注入 `Services`（不得在 Controller 內 `new`）
- 將「判斷/計算/規則」下推至 `Utilities`，Controller 僅負責組裝呼叫與回傳

**禁止 (MUST NOT)：**
- 直接呼叫 `Repositories`
- 直接處理資料庫交易或資料一致性規則（必須集中於 `Repositories` / `Utilities`）

### §4.5 Services

**職責**：組合呼叫的穩定操作入口；提供 Controller/Middleware/Filters 一個一致的呼叫介面。

**必須 (MUST)：**
- 只做「組合與轉交」：呼叫 `Repositories` 取得資料、呼叫 `Utilities` 執行判斷/計算、最後回傳結果
- 作為依賴注入的主要對外入口（Controller 不得越過 Service 直接使用 Repository）

**禁止 (MUST NOT)：**
- 在 `Services` 內實作業務規則判斷或複雜計算（必須下推至 `Utilities`）

### §4.6 Repositories

**職責**：資料庫互動與持久化邊界（查詢、寫入、交易、鎖定策略、EF 追蹤與映射）。

**必須 (MUST)：**
- 將所有資料存取集中於 Repository（Controller/Service/Utility 不得直接存取資料庫）
- 僅回傳 `Models`（或 `Core` 定義之共用純型別），不得回傳 Controller/HTTP 相關型別

**禁止 (MUST NOT)：**
- 呼叫 `Services`
- 將業務規則判斷塞回資料存取層（規則與判斷必須由 `Utilities` 負責）

### §4.7 Models

**職責**：資料型別定義（Entity/Record/DTO/Request/Response），承載資料結構，不承載流程與副作用。

**禁止 (MUST NOT)：**
- 在 Model 內直接進行資料庫存取
- 在 Model 內實作流程/規則（判斷與計算必須移至 `Utilities`）

### §4.8 Utilities

**職責**：重邏輯、功能、判斷與規則的集中地（例如驗證、計算、狀態轉換、規則表）。

**必須 (MUST)：**
- 所有「需要判斷與計算」的內容必須在此層實作，`Services` 僅組合呼叫
- 僅依賴 `Models` / `Core`（不得依賴 HTTP/MVC 具體型別）

### §4.9 Core

**職責**：跨功能、跨層可重用的核心能力（例如時間、序列化、共用常數/錯誤碼、通用結果型別）。

**禁止 (MUST NOT)：**
- 依賴 MVC / HTTP / Web 具體型別
- 依賴任何專案功能層的具體實作（Core 只能被依賴）

### §4.10 Views（Razor）

**職責**：畫面呈現；透過 `ViewModel`/`Model` 顯示資料與輸出 HTML。

**禁止 (MUST NOT)：**
- 在 View 內呼叫 `Services` / `Repositories` / `Utilities` / `Core`
- 在 View 內進行資料庫存取或寫入行為

---

## §5 程式碼規範

### §5.1 Ajax 使用規範

本專案前端 Ajax 統一採用 jQuery `$.ajax`，不得混用 `fetch`、Axios 或其他呼叫風格。

**必須 (MUST)：**
- 使用 `$.ajax({ url, type, data, success, error })` 標準格式。
- `GET` 僅用於讀取資料或載入 Partial View。
- `POST` 僅用於提交、刪除、修改等會改變狀態的操作。
- 成功後必須明確更新畫面（例如回填容器、重新查詢、關閉 Modal、重新初始化元件）。
- 失敗時必須顯示 `[失敗]` 開頭之訊息。

**正確：**

```javascript
$.ajax({
    url: '@Url.Action("SearchItems", "Sample")',
    type: 'GET',
    success: (html) => {
        $('#resultContainer').html(html);
    },
    error: () => {
        showToast('[失敗] 無法載入資料，請稍後再試', 'error');
    }
});
```

**禁止：**

```javascript
fetch('/Backoffice/Sample/SearchItems')
    .then(response => response.text())
    .then(html => $('#resultContainer').html(html));
```

### §5.2 Razor `@model` 使用規範

**必須 (MUST)：**
- 列表型 Partial View 使用 `@model List<T>`。
- 主頁面 View 僅負責組合 Partial、設定 `ViewData` / `Layout`，不得直接承接列表集合模型。
- Partial 若為可空列表輸入，輸出端必須容忍 `Model == null`。

**正確：**

```csharp
@model List<ItemViewModel>

@foreach (var item in Model ?? new List<ItemViewModel>())
{
    <tr>
        <td>@item.Name</td>
    </tr>
}
```

**禁止：**

```csharp
@model IEnumerable<ItemViewModel>
```

### §5.3 註解方式與內容

**必須 (MUST)：**
- `public` 與 `internal` 的類別、方法、屬性使用 `/// <summary>` 文件註解。
- 參數或回傳語意不明確時，補上 `<param>`、`<returns>`。
- 註解內容只描述責任、輸入、輸出、限制，不描述方法體內逐步流程。

**正確：**

```csharp
/// <summary>
/// 查詢項目列表。
/// </summary>
/// <param name="scopeIds">範圍 ID 過濾條件。</param>
public List<ItemModel> QueryItemList(List<int>? scopeIds = null)
    => ValidItems
        .Where(x => scopeIds == null || !scopeIds.Any() || (x.ScopeId.HasValue && scopeIds.Contains(x.ScopeId.Value)))
        .ToList();
```

**禁止：**

```csharp
public List<ItemModel> QueryItemList(List<int>? scopeIds = null)
{
    // 先查詢資料
    // 再做範圍過濾
    return ValidItems.ToList();
}
```

### §5.4 端點文件化

Controller Action 屬於對外契約（HTTP API），必須可由程式碼直接判讀其輸入、輸出與可能回應。

**必須 (MUST)：**
- 每個 Action 必須明確標示輸入來源（例如 `[FromRoute]`、`[FromQuery]`、`[FromBody]`），不得讓繫結行為依賴猜測。
- 每個 Action 必須標示回應型別與狀態碼（例如 `[ProducesResponseType]`），讓 API 行為可機械判讀。
- 任何驗證規則必須以框架機制表達（例如 DataAnnotations 或專案既定驗證方式），不得散落於 Controller 內手寫判斷。

**正確：**

```csharp
[HttpGet("{id:long}")]
[ProducesResponseType(typeof(WorkspaceModel), StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status404NotFound)]
public IActionResult QueryWorkspace([FromRoute] long id)
{
    var model = _services.QueryWorkspace(id);
    return model == null ? NotFound() : Ok(model);
}
```

**禁止：**

```csharp
[HttpGet("{id}")]
public IActionResult QueryWorkspace(long id)
{
    if (id <= 0) return BadRequest("id error");
    return Ok(_services.QueryWorkspace(id));
}
```

### §5.5 注釋規則

- 禁止：在方法體內用行內注釋說明邏輯步驟（方法體應自我說明）
- 禁止：用 `/* */` 區塊注釋
- 禁止：注釋說明顯而易見的程式碼（如 `// 設定為 true`）

---

## §6 合規判定

下列情況視為不合規，必須修正。

| 編號 | 違規情況 |
| --- | --- |
| V-01 | `Views` 直接呼叫 `Services`、`Repositories`、`Utilities`、`Core` |
| V-02 | `Controllers` 直接呼叫 `Repositories` 或直接操作資料庫 |
| V-03 | `Controllers` 內出現業務規則判斷、複雜計算或資料一致性控制 |
| V-04 | `Services` 內實作業務規則判斷或複雜計算，未下推至 `Utilities` |
| V-05 | `Repositories` 依賴 `Services`，或於 Repository 內呼叫 Service |
| V-06 | `Models` 內含業務邏輯、I/O 操作或資料庫存取 |
| V-07 | `Utilities` 或 `Core` 依賴 MVC / HTTP / Web 具體型別 |
| V-08 | `Middleware` 或 `Filters` 內直接存取資料庫或承載業務規則 |
| V-09 | 函式名稱未依層使用既定前綴，或跨層混用前綴（如 `Services` 使用 `Query`、`Repositories` 使用 `Load`） |
| V-10 | 層內檔案後綴不符規定（如 `*Service.cs` 置於 `Services`、`*Repository.cs` 置於 `Repositories`） |
| V-11 | 同功能主題在同一層拆分為多個檔案 |
| V-12 | `Models`、`Services`、`Repositories`、`Utilities`、`Core` 內出現以主題為名的子資料夾，用以規避單一檔案限制 |
| V-13 | 前端 Ajax 未使用 jQuery `$.ajax` 標準格式，或混用 `fetch`、Axios |
| V-14 | 列表型 Partial View 未使用 `@model List<T>` |
| V-15 | `public` / `internal` 類別、方法、屬性缺少 `/// <summary>` 文件註解 |
| V-16 | 方法體內出現邏輯說明的行內註解 |
| V-17 | Controller Action 未明確標示輸入來源或回應型別，導致端點契約不可機械判讀 |
| V-18 | 未經開發者明確指示，將單一 `.csproj` 拆分為多個專案 |
