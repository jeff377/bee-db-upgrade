# BeeDbUpgrade

Bee.NET 資料庫升級工具 — 用於升級資料庫結構，使其與定義檔一致。

## 需求

- .NET 10.0 (Windows)
- Bee.NET NuGet 套件 v3.6.2+

## 建置

```bash
dotnet restore
dotnet build --configuration Release
```

## 發布

```bash
dotnet publish src/BeeDbUpgrade/BeeDbUpgrade.csproj -c Release -r win-x64
```

產出位於 `src/BeeDbUpgrade/bin/Release/net10.0-windows/publish/win-x64/`

## 授權

MIT License
