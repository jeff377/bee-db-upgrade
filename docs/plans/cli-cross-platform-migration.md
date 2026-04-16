# BeeDbUpgrade 跨平台 CLI 改造計畫

## 1. 背景與目標

目前 `BeeDbUpgrade` 為 .NET 10 WinForms 工具，僅可在 Windows 執行。專案目標是讓本工具同時支援 **Windows 與 macOS**（Linux 免費取得）。

經評估三種候選方案（MAUI / Avalonia UI / CLI），最終採用 **CLI 方案**，理由如下：

- 工具本質為流程性作業（選資料庫 → 執行升級 → 輸出記錄），GUI 非必要。
- 相較 MAUI 與 Avalonia，改寫成本最低、部署檔案最小、無 UI 框架鎖定。
- 可直接整合至 CI/CD、排程（cron / Task Scheduler），提升可維運性。
- 使用 Apple Silicon 原生執行（`osx-arm64`），無 Rosetta 開銷。

## 2. 相依套件現況

| 套件 | 版本 | TFM | 跨平台 |
|---|---|---|---|
| `Microsoft.Data.SqlClient` | 6.1.3 | netstandard2.0 / net8.0+ | ✅ |
| `Bee.Business` | 3.6.2+ | net10.0 | ✅ |
| `Bee.Cache` | 3.6.2+ | net10.0 | ✅ |
| `Bee.Repository` | 3.6.2+ | net10.0 | ✅ |
| ~~`Bee.UI.WinForms`~~ | — | net10.0-windows | ❌（CLI 版移除） |

> 備註：Bee.* 核心套件已調整為 `net10.0`（無 `-windows`），可在 macOS / Linux 執行。

## 3. 設計決策（已確認）

| 項目 | 決策 |
|---|---|
| 操作模式 | **互動式 + 參數雙模式**：不帶參數時進入互動選單；帶 `--db` / 其他參數時直接執行 |
| 終端 UI 函式庫 | **Spectre.Console**（提供選單、進度條、彩色輸出；三平台皆支援） |
| 專案結構 | **單一專案**，維持 `BeeDbUpgrade.csproj`，不額外拆出 Core |
| Log 行為 | **總是自動儲存**到當前工作目錄，檔名 `DbToolsLog_yyyyMMdd_HHmmss.txt` |

## 4. 目標 CLI 介面草案

### 4.1 互動模式（不帶必要參數）

```
$ BeeDbUpgrade
連線方式：Local
? 請選擇要升級的資料庫: (使用 ↑↓ 鍵)
  > MyDB_Prod
    MyDB_Test
    MyDB_Dev

升級中 [##########          ]  45% | 23/51 | Customer
[OK]   Customer        : 結構升級
[--]   Product         : 結構一致
...

執行完畢，共升級 5 個資料表
Log 已儲存至：./DbToolsLog_20260416_143022.txt
```

### 4.2 參數模式（自動化）

```
$ BeeDbUpgrade --db MyDB_Prod
$ BeeDbUpgrade --db MyDB_Prod --endpoint /path/to/DefinePath
$ BeeDbUpgrade --list                 # 列出可用資料庫
$ BeeDbUpgrade --help
```

### 4.3 參數清單

| 參數 | 簡寫 | 型態 | 說明 |
|---|---|---|---|
| `--db <name>` | `-d` | string | 指定要升級的資料庫名稱；省略則進入互動選單 |
| `--endpoint <path>` | `-e` | string | 指定 DefinePath（對應目前 launchSettings 的 `--Endpoint`） |
| `--list` | `-l` | flag | 僅列出可用資料庫清單後退出 |
| `--log-dir <path>` | | string | 指定 log 輸出目錄（預設為當前目錄） |
| `--no-color` | | flag | 關閉彩色輸出（適合 CI log） |
| `--help` | `-h` | flag | 顯示說明 |
| `--version` | `-v` | flag | 顯示版本 |

## 5. 檔案層級變更計畫

### 5.1 修改

#### `src/BeeDbUpgrade/BeeDbUpgrade.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>                    <!-- 原本 WinExe -->
    <TargetFramework>net10.0</TargetFramework>       <!-- 原本 net10.0-windows -->
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <RootNamespace>DbUpgrade</RootNamespace>
    <AssemblyName>BeeDbUpgrade</AssemblyName>
    <SatelliteResourceLanguages>en</SatelliteResourceLanguages>
    <!-- 移除 <UseWindowsForms> -->
    <!-- 移除 <ApplicationIcon> 或改用跨平台處理方式 -->
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.Data.SqlClient" Version="6.1.3" />
    <PackageReference Include="Bee.Business" Version="3.6.2" />
    <PackageReference Include="Bee.Cache" Version="3.6.2" />
    <PackageReference Include="Bee.Repository" Version="3.6.2" />
    <!-- 移除 Bee.UI.WinForms -->
    <!-- 新增： -->
    <PackageReference Include="Spectre.Console" Version="0.49.1" />
    <PackageReference Include="Spectre.Console.Cli" Version="0.49.1" />
  </ItemGroup>

  <!-- Release 組態、Publish 後清理 XML/PDB 等既有設定保留 -->
</Project>
```

> 說明：`Spectre.Console.Cli` 提供指令解析；也可改用 `System.CommandLine`，但 Spectre 整合度更好。

#### `src/BeeDbUpgrade/Common/AppInfo.cs`

- 移除 `using Bee.UI.WinForms;`
- 將 `new UIViewService()` 替換為 `new ConsoleViewService()`（見 5.2 新增檔案）
- 其餘初始化邏輯保持不變（`BackendInfo.*`、`RepositoryInfo.*`、`DbProviderManager.RegisterProvider`）

> **待確認**：`Bee.UI.Core` 是否已內建跨平台 `IViewService` 實作（如 `DefaultViewService` / `NullViewService`）。若有，直接使用；若無，自行實作（5.2）。

#### `src/BeeDbUpgrade/Program.cs`（全面重寫）

```csharp
using Spectre.Console.Cli;

namespace DbUpgrade;

internal static class Program
{
    static int Main(string[] args)
    {
        var app = new CommandApp<UpgradeCommand>();
        app.Configure(config =>
        {
            config.SetApplicationName("BeeDbUpgrade");
            config.AddCommand<UpgradeCommand>("upgrade").IsDefault();
            config.AddCommand<ListCommand>("list");
        });
        return app.Run(args);
    }
}
```

#### `README.md`

- 改寫需求段落（支援 Windows / macOS / Linux）
- 新增互動模式與參數模式使用範例
- 調整 publish 指令，列出各平台 RID

#### `.github/workflows/release-BeeDbUpgrade.yml`

- 改為多平台矩陣建置（`win-x64` / `osx-arm64` / `osx-x64` / `linux-x64`）
- 每個平台各產出一份 zip / tar.gz 上傳至 Release
- runner 改為 `ubuntu-latest`（或維持 `windows-latest`，但需使用 bash）
- 若保留 `FolderProfile.pubxml`，改為每平台一份 profile

#### `src/BeeDbUpgrade/Properties/PublishProfiles/FolderProfile.pubxml`

- `TargetFramework` 改為 `net10.0`
- 視需要新增 `FolderProfile-OsxArm64.pubxml`、`FolderProfile-OsxX64.pubxml`、`FolderProfile-LinuxX64.pubxml`

#### `src/BeeDbUpgrade/Properties/launchSettings.json`

- `--Endpoint` 改為小寫統一慣例：`--endpoint`（與 Spectre.Console.Cli 一致）

### 5.2 新增

#### `src/BeeDbUpgrade/Common/ConsoleViewService.cs`（視待確認結果決定）

負責實作 `Bee.UI.Core.IViewService` 的跨平台版本，將訊息輸出改為 `AnsiConsole`：

- `MsgBox` / `InfoMessage` → `AnsiConsole.MarkupLine("[green]{0}[/]", msg)`
- `ErrorMsgBox` → `AnsiConsole.MarkupLine("[red]{0}[/]", msg)`
- `Confirm` → `AnsiConsole.Confirm(msg)`

#### `src/BeeDbUpgrade/Commands/UpgradeCommand.cs`

Spectre.Console.Cli 的 `Command<TSettings>` 類別，包含：

- 參數定義（`[CommandOption("-d|--db <DB>")]` 等）
- 若 `--db` 為空 → 呼叫 `AnsiConsole.Prompt(new SelectionPrompt<DatabaseItem>()...)` 進入互動選單
- 呼叫 `UpgradeRunner.Run(dbName)`
- 使用 `AnsiConsole.Progress` 顯示進度條
- 結尾將 log 寫入 `DbToolsLog_yyyyMMdd_HHmmss.txt`

#### `src/BeeDbUpgrade/Commands/ListCommand.cs`

輸出可用資料庫清單（表格格式）後直接退出。

#### `src/BeeDbUpgrade/UpgradeRunner.cs`

從 `frmMainForm.cs` 抽出的核心邏輯（純業務邏輯，無 UI 依賴）：

```csharp
public sealed class UpgradeRunner
{
    public event Action<UpgradeProgress>? ProgressChanged;
    public UpgradeResult Run(string dbName) { ... }
}

public record UpgradeProgress(int Current, int Total, string TableName);
public record UpgradeResult(int UpgradedCount, int TotalCount, IReadOnlyList<string> Lines);
```

### 5.3 刪除

- `src/BeeDbUpgrade/frmMainForm.cs`
- `src/BeeDbUpgrade/frmMainForm.Designer.cs`
- `src/BeeDbUpgrade/frmMainForm.resx`
- `src/BeeDbUpgrade/BeeDbUpgrade.ico`（CLI 不需要 app icon；或保留作為套件識別）

## 6. 待確認事項

1. **`Bee.UI.Core.IViewService` 介面規格**
   - 需要知道介面成員才能實作 `ConsoleViewService`
   - 若 `Bee.UI.Core` 已有預設（跨平台）實作，直接用之
   - **等 Bee.* 套件正式發佈後再行確認**

2. **`ClientInfo.Initialize` 是否支援無 ViewService 初始化**
   - 若有 overload 接受 `null`，則連 `ConsoleViewService` 都不必實作

3. **`UIFunc.GetConnectText()` 的替代方案**
   - `frmMainForm_Load` 呼叫 `UIFunc.GetConnectText()` 取得連線方式文字
   - 確認 `Bee.UI.Core` 是否有同等 API，或需從 `ClientInfo` / `BackendInfo` 直接讀取

4. **`--Endpoint` 參數的處理位置**
   - 目前 `launchSettings.json` 傳 `--Endpoint`，推測由 `AppInfo.Initialize()` 前某處讀取
   - 需確認由 Bee.* 框架解析，還是要在 CLI 端自行解析後注入

## 7. 建置與發佈

### 7.1 本機建置

```bash
dotnet restore
dotnet build -c Release
```

### 7.2 各平台單檔發佈

```bash
# Windows x64
dotnet publish src/BeeDbUpgrade/BeeDbUpgrade.csproj \
  -c Release -r win-x64 --self-contained true \
  -p:PublishSingleFile=true -p:IncludeNativeLibrariesForSelfExtract=true

# macOS Apple Silicon
dotnet publish src/BeeDbUpgrade/BeeDbUpgrade.csproj \
  -c Release -r osx-arm64 --self-contained true \
  -p:PublishSingleFile=true -p:IncludeNativeLibrariesForSelfExtract=true

# macOS Intel
dotnet publish src/BeeDbUpgrade/BeeDbUpgrade.csproj \
  -c Release -r osx-x64 --self-contained true \
  -p:PublishSingleFile=true -p:IncludeNativeLibrariesForSelfExtract=true

# Linux x64（附加）
dotnet publish src/BeeDbUpgrade/BeeDbUpgrade.csproj \
  -c Release -r linux-x64 --self-contained true \
  -p:PublishSingleFile=true -p:IncludeNativeLibrariesForSelfExtract=true
```

### 7.3 GitHub Actions 矩陣建置

```yaml
strategy:
  matrix:
    include:
      - os: windows-latest
        rid: win-x64
        ext: zip
      - os: macos-latest
        rid: osx-arm64
        ext: tar.gz
      - os: macos-13       # Intel runner
        rid: osx-x64
        ext: tar.gz
      - os: ubuntu-latest
        rid: linux-x64
        ext: tar.gz
```

每個 job 產出 `BeeDbUpgrade-<version>-<rid>.<ext>` 並附加至 Release。

## 8. 相容性與風險

| 風險 | 說明 | 對策 |
|---|---|---|
| macOS 上的 SQL Server 連線 | `Microsoft.Data.SqlClient` 在 macOS 的 Kerberos/整合式驗證有限 | 改用 SQL Auth；若需 Kerberos 再評估 |
| 檔案路徑分隔符 | `--endpoint` 路徑在 Windows 使用 `\`，macOS 使用 `/` | 統一呼叫 `Path.Combine` / `Path.DirectorySeparatorChar` |
| 終端色彩 | 部分 CI log 不支援 ANSI 跳脫 | 提供 `--no-color` 選項並偵測 `NO_COLOR` 環境變數 |
| Bee.* 套件跨平台驗證 | 雖已改為 `net10.0`，仍需實機測試 | 規劃 macOS 煙霧測試清單（連線、讀取 Schema、執行 Upgrade） |
| ICO 檔案 | `.ico` 為 Windows 格式，macOS/Linux 不使用 | 從 csproj 移除 `<ApplicationIcon>`；檔案可保留作為文件素材 |

## 9. 後續步驟

1. **等 Bee.* 套件正式發佈含 `net10.0` TFM 的版本後**，重新打開本計畫。
2. 先在 macOS 建立煙霧測試專案（`dotnet new console` + `Bee.Business`）驗證可載入與呼叫 `ClientInfo.SystemApiConnector.ExecFunc`。
3. 依 §5 逐一實作。
4. 建置並於三平台本機測試（至少 `win-x64` 與 `osx-arm64`）。
5. 調整 CI workflow 產出三平台 artifact。
6. 更新 `README.md` 與 Release Notes。

---

**文件版本**：2026-04-16 初版
**分支**：`claude/evaluate-maui-crossplatform-EzJhf`
