# Implementation Plan: 基本 RBAC 權限系統

**Branch**: `[001-basic-rbac]` | **Date**: 2025-12-03 | **Spec**: `specs/basic-rbac/spec.md`  
**Input**: Feature specification from `/specs/basic-rbac/spec.md`

## Summary

實作一個以 org 為邊界的基本 RBAC 系統，支援 `user` / `org` / `role` / `permission` 與對應關聯，  
在每個 org 內提供三種角色（`owner` / `admin` / `user`）以及至少 CRUD 四種動作的 permission，  
並透過一個抽象的 RBAC 介面（provider）供 RESTful API 與應用服務查詢授權結果。  
技術棧採用 **Golang + MongoDB + Redis + RESTful API**，以測試優先（包含單元測試與整合測試）為前提，  
新功能對應範圍內的核心邏輯以實質可測試單元為主，力求達成接近 100% 的行覆蓋率。  

## Technical Context

**Language/Version**: Go 1.22（可視實際環境微調）  
**Primary Dependencies**:  
- Web/REST: `net/http` 或輕量框架（例如 `chi`），避免過度設計。  
- MongoDB driver: `go.mongodb.org/mongo-driver`。  
- Redis client: `github.com/redis/go-redis/v9` 用於快取權限查詢結果（如有需要）。  
**Storage**:  
- 主資料庫：MongoDB（儲存 user / org / role / permission / 關聯）。  
- 快取：Redis（可選，先以簡單實作為主，必要時再導入快取）。  
**Testing**:  
- 單元測試：Go `testing` + `testify`（斷言與 mock 幫手），覆蓋 service、provider、repository。  
- 整合測試：使用 in-memory MongoDB（或測試專用資料庫）、docker-compose 環境，驗證 API 行為與授權判斷。  
**Target Platform**: Linux server / 容器化部署（例如 Docker）。  
**Project Type**: 單一後端服務（single backend service）。  
**Performance Goals**:  
- 單次 RBAC 權限檢查延遲目標 < 5ms（不含網路 I/O），以 in-memory 或快取結果為主。  
**Constraints**:  
- 測試優先，對此功能的核心邏輯（RBAC provider + service）目標為 100% 行覆蓋率；  
  但對於基礎設施（例如啟動程式、錯誤處理樣板）可接受較低覆蓋率，只要不影響品質。  
**Scale/Scope**:  
- 目標先支援單一服務實例與中小規模 org/user 數量，後續擴充與 sharding 策略視實際需求在後續迭代定義。  

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **可測試性**：  
  - 規格已為三個 User Story 定義獨立測試策略（單元／整合），  
  - 計畫中各層（provider、service、repository、API handler）皆有明確可測單元，  
  - 將以 Go `testing` 搭配 mock/in-memory 實作達成高覆蓋率，關鍵邏輯以 100% 為目標。  
- **MVP 切分**：  
  - P1（建立 org + owner）為首個 MVP，可獨立實作與驗證；  
  - P2（org 內角色與 CRUD 權限）與 P3（抽象 RBAC 介面）可在 P1 完成後增量實作，各自有獨立測試與驗證路徑。  
- **簡單優先**：  
  - 採單一後端服務與單一 Go module，避免一開始就切多專案或多服務；  
  - 資料存取層以直接 repository + Mongo collection 模型為主，不額外引入過度抽象的 ORM。  
- **一致性**：  
  - 遵守憲章的命名與層次結構，將 RBAC 邏輯集中在 `internal/rbac`，  
  - REST API 與服務層不直接操作 Mongo/Redis，而是透過抽象 provider/service，利於未來替換實作。  

## Project Structure

### Documentation (this feature)

```text
specs/basic-rbac/
├── plan.md              # 本檔案（/speckit.plan）
├── research.md          # Phase 0：技術調研與對比
├── data-model.md        # Phase 1：Mongo 資料模型與關聯
├── quickstart.md        # Phase 1：如何啟動服務與呼叫 RBAC API
├── contracts/           # Phase 1：RESTful API 規格（例如 OpenAPI）
└── tasks.md             # Phase 2：/speckit.tasks 產生的任務列表
```

### Source Code (repository root)

```text
cmd/
└── rbac-server/                 # 服務進入點（main），組裝 HTTP server、DI

internal/
├── rbac/                        # 核心 RBAC 邏輯（不依賴具體傳輸層）
│   ├── domain/                  # 實體與介面定義（User, Org, Role, Permission 等）
│   ├── service/                 # Use case / application service（指派角色、檢查權限）
│   └── provider/                # 抽象 RBAC Provider 介面與預設實作
├── mongo/                       # MongoDB 連線與 repository 實作
│   ├── client.go
│   └── rbac_repository.go
├── redis/                       # Redis 客戶端與（選用）快取層
│   └── client.go
└── http/                        # RESTful API handler 與 router
    ├── router.go
    └── handlers/
        ├── org_handler.go
        ├── role_handler.go
        └── permission_handler.go

tests/
├── contract/                    # 針對 REST API 的契約測試（可使用 httptest）
├── integration/                 # 整合測試（含 Mongo/Redis 測試環境）
└── unit/                        # 單元測試（rbac service、provider 等）
```

**Structure Decision**:  
- 採用單一後端專案結構，以 Go 常見的 `cmd/` + `internal/` + `tests/` 佈局，  
- RBAC 相關邏輯集中在 `internal/rbac`，以介面抽象與聚焦 domain/service 層，  
- HTTP 與資料庫適配層分別放在 `internal/http` 與 `internal/mongo` / `internal/redis`，  
- 測試則依照憲章建議，區分 `unit` / `integration` / `contract` 三種層級。  

## Complexity Tracking

> **目前無違反憲章必須特別說明的複雜度決策。**

若後續在設計或實作中引入額外模式（例如 event-based 授權快取失效、跨服務 RBAC 同步機制等），  
需在此表中記錄違反「簡單優先」原則的具體決策與拒絕較簡單方案的原因。  

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|--------------------------------------|
| *(暫無)*  | *(暫無)*   | *(暫無)*                             |


