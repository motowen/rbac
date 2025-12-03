---

description: "Task list for 基本 RBAC 權限系統"
---

# Tasks: 基本 RBAC 權限系統

**Input**: Design documents from `/specs/basic-rbac/`  
**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/

**Tests**: 本功能採「測試優先」，對核心 RBAC 邏輯（provider/service/repository）目標為接近 100% 行覆蓋率。各 User Story 區段都會先列出測試任務，再列實作任務。

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story,
and to align with the constitution's MVP/independent slice principle.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- 單一後端專案：`cmd/`, `internal/`, `tests/` 於 repository root。  
- REST API：`internal/http` 下的 router 與 handlers。  
- RBAC Domain / Service / Provider：`internal/rbac/...`。  
- Mongo / Redis：`internal/mongo`, `internal/redis`。  

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: 專案初始化與基本目錄結構建立

- [ ] T001 建立 Go module 與基礎目錄結構（`go mod init`、建立 `cmd/`, `internal/`, `tests/`）  
  - 路徑：`go.mod`, `cmd/rbac-server/main.go`, `internal/`, `tests/`  
- [ ] T002 新增基礎依賴（HTTP router, Mongo driver, Redis client, testify 等）  
  - 路徑：`go.mod`  
- [ ] T003 [P] 建立 RBAC 核心目錄結構與空檔案（僅留待實作的骨架）  
  - 路徑：`internal/rbac/domain/`, `internal/rbac/service/`, `internal/rbac/provider/`  
- [ ] T004 [P] 建立 Mongo/Redis 客戶端與設定檔骨架（尚未實作細節）  
  - 路徑：`internal/mongo/client.go`, `internal/redis/client.go`, `internal/config/config.go`  
- [ ] T005 [P] 初始化 HTTP server 與 router 骨架（尚未實作 RBAC 邏輯）  
  - 路徑：`internal/http/router.go`, `cmd/rbac-server/main.go`  
- [ ] T006 [P] 設定基本 Makefile 或 task runner（包含 `test` 指令與 coverage 報告）  
  - 路徑：`Makefile` 或 `Taskfile.yml`  

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: 在任何 User Story 開始前，建立穩定的資料存取與 RBAC 抽象層

**⚠️ CRITICAL**: 完成本階段前不得開始任何 US1/US2/US3 相關實作

- [ ] T007 定義 RBAC 核心 domain 結構（User, Org, Role, Permission, UserRole, RolePermission）  
  - 路徑：`internal/rbac/domain/entities.go`  
- [ ] T008 定義 RBAC 抽象介面（例如 `RbacProvider` 與 service interface）  
  - 路徑：`internal/rbac/provider/provider.go`, `internal/rbac/service/service.go`  
- [ ] T009 設計並實作 Mongo 資料模型與集合名稱（含索引需求的 TODO 註解）  
  - 路徑：`internal/mongo/rbac_repository.go`  
- [ ] T010 建立 RBAC repository 介面與 Mongo 實作（CRUD for org/role/permission/user-role/role-permission）  
  - 路徑：`internal/rbac/domain/repository.go`, `internal/mongo/rbac_repository.go`  
- [ ] T011 實作基礎 Mongo 連線與設定載入（含測試用連線選項）  
  - 路徑：`internal/mongo/client.go`, `internal/config/config.go`  
- [ ] T012 [P] 建立 Redis 客戶端骨架與 RBAC 快取介面（先以 no-op 實作，之後再決定是否啟用）  
  - 路徑：`internal/redis/client.go`, `internal/rbac/provider/cache.go`  
- [ ] T013 將 HTTP 層與 RBAC provider/service 透過 DI / 組裝函式串接  
  - 路徑：`cmd/rbac-server/main.go`, `internal/http/router.go`  
- [ ] T014 建立基礎單元測試架構與 helper（對 repository/service/provider 的測試基類或共用 setup）  
  - 路徑：`tests/unit/rbac_test_helper.go`, `tests/integration/test_helper.go`  

**Checkpoint**:  
- RBAC domain / repository / provider 介面已定義，Mongo 連線與 repository 骨架完成，  
- 測試環境 helper 就緒，可以開始針對各 User Story 撰寫測試與實作。  

---

## Phase 3: User Story 1 - 建立組織並指派 Owner (Priority: P1) 🎯 MVP

**Goal**: 讓系統能建立 org 並指定一位 user 為該 org 的 owner，只有 owner 可以修改 org 基本資訊。  
**Independent Test**: 僅啟用與 org 建立與 owner 指派相關的 API / service，即可完整驗證本故事。  

### Tests for User Story 1 (測試優先，必填) ⚠️

- [ ] T015 [P] [US1] 撰寫 RBAC service 單元測試：建立 org 並指派 owner  
  - 路徑：`tests/unit/rbac_service_us1_test.go`  
- [ ] T016 [P] [US1] 撰寫 RBAC repository 單元測試：儲存與讀取 org 與 user-role 關聯  
  - 路徑：`tests/unit/rbac_repository_us1_test.go`  
- [ ] T017 [P] [US1] 撰寫 REST API 契約測試：建立 org + 指派 owner 的 HTTP 介面  
  - 路徑：`tests/contract/http_org_us1_test.go`  
- [ ] T018 [US1] 撰寫整合測試：從 API → service → repository → Mongo 完整流程（成功與權限拒絕）  
  - 路徑：`tests/integration/rbac_us1_integration_test.go`  

### Implementation for User Story 1

- [ ] T019 [P] [US1] 在 RBAC service 實作「建立 org 並指派 owner」的 use case  
  - 路徑：`internal/rbac/service/org_service.go`  
- [ ] T020 [P] [US1] 在 repository 實作對 org 與 user-role 的新增與查詢（僅 US1 需要的欄位與操作）  
  - 路徑：`internal/mongo/rbac_repository.go`  
- [ ] T021 [P] [US1] 實作 REST handler：建立 org 並指派 owner 的 endpoint  
  - 路徑：`internal/http/handlers/org_handler.go`  
- [ ] T022 [US1] 在 router 新增 US1 相關 route（例如 POST `/orgs`）  
  - 路徑：`internal/http/router.go`  
- [ ] T023 [US1] 更新錯誤處理與回應格式（成功/失敗、權限不足）  
  - 路徑：`internal/http/handlers/org_handler.go`, 共用錯誤型別檔案  
- [ ] T024 [US1] 確認所有 US1 相關單元/整合/契約測試通過並檢查覆蓋率報告  
  - 路徑：`Makefile` 或 CI 設定檔、coverage 報告檔（例如 `coverage.out`）  

**Checkpoint**:  
- 能透過 API 建立 org 並指派 owner，  
- owner 可以成功修改 org 基本資訊，其他 user 嘗試修改時會被拒絕，  
- 所有 US1 測試通過，核心邏輯行覆蓋率達到預期標準。  

---

## Phase 4: User Story 2 - 在單一 org 中管理角色與 CRUD 權限 (Priority: P2)

**Goal**: 在單一 org 內，owner 可以指派 admin/user 並設定 CRUD 權限，系統依角色與 permission 判斷是否允許操作。  
**Independent Test**: 在單一 org 下，建立一組 owner/admin/user，僅啟用 P2 相關 API，即可完整驗證權限控制。  

### Tests for User Story 2 (測試優先，必填) ⚠️

- [ ] T025 [P] [US2] 撰寫 RBAC service 單元測試：在 org 內指派 `admin` / `user` 角色  
  - 路徑：`tests/unit/rbac_service_us2_roles_test.go`  
- [ ] T026 [P] [US2] 撰寫 RBAC service 單元測試：設定 `project` 類資源的 CRUD permission 映射  
  - 路徑：`tests/unit/rbac_service_us2_permissions_test.go`  
- [ ] T027 [P] [US2] 撰寫 RBAC provider 單元測試：依 user 在 org 的角色集合決定 `can(user, org, resource, action)` 結果  
  - 路徑：`tests/unit/rbac_provider_us2_test.go`  
- [ ] T028 [P] [US2] 撰寫 REST API 契約測試：角色指派與 CRUD 權限設定相關的 endpoints  
  - 路徑：`tests/contract/http_roles_permissions_us2_test.go`  
- [ ] T029 [US2] 撰寫整合測試：驗證 admin 可以建立/更新 project，而 user 無法刪除 project  
  - 路徑：`tests/integration/rbac_us2_integration_test.go`  

### Implementation for User Story 2

- [ ] T030 [P] [US2] 擴充 RBAC domain：增加資源型別與動作的結構（例如 `Resource`, `Action` 常數）  
  - 路徑：`internal/rbac/domain/entities.go`  
- [ ] T031 [P] [US2] 在 RBAC service 實作角色指派與 CRUD permission 綁定的 use case  
  - 路徑：`internal/rbac/service/role_service.go`, `internal/rbac/service/permission_service.go`  
- [ ] T032 [P] [US2] 在 repository 實作 RolePermission 與 UserRole 的存取與查詢  
  - 路徑：`internal/mongo/rbac_repository.go`  
- [ ] T033 [P] [US2] 實作/擴充 REST handlers：角色指派與 CRUD 權限設定 endpoints  
  - 路徑：`internal/http/handlers/role_handler.go`, `internal/http/handlers/permission_handler.go`  
- [ ] T034 [US2] 在 router 新增 P2 相關 route（例如 POST `/orgs/{id}/roles`, `/orgs/{id}/permissions`）  
  - 路徑：`internal/http/router.go`  
- [ ] T035 [US2] 擴充錯誤處理與審計資訊（例如記錄誰修改了哪個 role/permission）  
  - 路徑：`internal/http/handlers/role_handler.go`, 共用 log/trace 模組  
- [ ] T036 [US2] 確認所有 US2 測試通過並檢查覆蓋率報告（核心授權邏輯達高覆蓋率）  
  - 路徑：coverage 報告檔（例如 `coverage.out`）  

**Checkpoint**:  
- 在單一 org 內可正確指派 owner/admin/user，  
- 可設定 `project` 類資源的 CRUD 權限，  
- 權限檢查符合 spec 提出之情境，測試通過且覆蓋率符合預期。  

---

## Phase 5: User Story 3 - 透過抽象 RBAC 介面進行權限查詢 (Priority: P3)

**Goal**: 讓應用服務只依賴抽象 RBAC 介面（provider），可在不改變呼叫端的前提下替換底層實作。  
**Independent Test**: 以 in-memory 實作與 Mongo/Redis 實作分別作為 provider，對同一組資料產生一致授權結果。  

### Tests for User Story 3 (測試優先，必填) ⚠️

- [ ] T037 [P] [US3] 撰寫 in-memory RBAC provider 單元測試（不依賴 Mongo/Redis）  
  - 路徑：`tests/unit/rbac_provider_inmemory_us3_test.go`  
- [ ] T038 [P] [US3] 撰寫 provider 介面一致性測試：同一情境下 in-memory 與 Mongo 實作結果一致  
  - 路徑：`tests/unit/rbac_provider_consistency_us3_test.go`  
- [ ] T039 [P] [US3] 撰寫使用 RBAC provider 的應用服務單元測試（例如虛擬的 `ProjectService`）  
  - 路徑：`tests/unit/project_service_us3_test.go`  
- [ ] T040 [US3] 撰寫整合測試：切換 provider 實作時，對 API 行為無感（只要設定來源相同，結果一致）  
  - 路徑：`tests/integration/rbac_us3_integration_test.go`  

### Implementation for User Story 3

- [ ] T041 [P] [US3] 實作 in-memory RBAC provider（供測試與開發環境使用）  
  - 路徑：`internal/rbac/provider/inmemory_provider.go`  
- [ ] T042 [P] [US3] 調整現有 RBAC service/provider 介面以支援多實作並透過設定選擇實作  
  - 路徑：`internal/rbac/provider/provider.go`, `internal/config/config.go`  
- [ ] T043 [P] [US3] 建立使用 RBAC provider 的範例應用服務（例如 `ProjectService` 的 `CanUpdateProject`）  
  - 路徑：`internal/rbac/service/example_project_service.go`  
- [ ] T044 [US3] 在組裝程式中加入 provider 選擇邏輯（環境變數或設定檔切換 in-memory / Mongo）  
  - 路徑：`cmd/rbac-server/main.go`  
- [ ] T045 [US3] 確認 US3 測試通過並檢查 provider 間一致性與覆蓋率  
  - 路徑：coverage 報告檔（例如 `coverage.out`）  

**Checkpoint**:  
- 應用層僅依賴 RBAC 抽象介面，  
- 可在不改變呼叫端程式碼的前提下替換 provider 實作，  
- 測試證明不同實作在相同資料下有一致授權結果。  

---

## Phase N: Polish & Cross-Cutting Concerns

**Purpose**: 影響多個 User Story 的橫切改進

- [ ] T046 [P] 補齊與整理文件（spec, plan, quickstart, contracts 中 RBAC 相關章節）  
  - 路徑：`specs/basic-rbac/quickstart.md`, `specs/basic-rbac/contracts/`  
- [ ] T047 改善錯誤訊息、log 與追蹤資訊（包含 orgId/userId/role 資訊）  
  - 路徑：`internal/http/handlers/`, 共用 log 模組  
- [ ] T048 整理與重構 RBAC service/provider/repository 中重複邏輯，保持簡單與可讀性  
  - 路徑：`internal/rbac/**`  
- [ ] T049 [P] 補充額外單元測試，針對過去 bug-prone 邏輯強化覆蓋率  
  - 路徑：`tests/unit/**`  
- [ ] T050 安排簡單效能測試或基準測試（benchmark）以確認 RBAC 權限查詢延遲在可接受範圍內  
  - 路徑：`tests/benchmark/rbac_benchmark_test.go`（如需要可新增此目錄）  

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: 無前置，可立即開始。  
- **Foundational (Phase 2)**: 依賴 Phase 1 完成，阻擋所有 User Story 實作。  
- **User Stories (Phase 3+)**: 均依賴 Foundational 完成後才能開始：  
  - US1（P1）優先，作為最小可行 MVP。  
  - US2、US3 可在 US1 完成後，以平行或序列方式實作。  
- **Polish (Final Phase)**: 依賴所有目標 User Story 完成後進行。  

### User Story Dependencies

- **User Story 1 (P1)**: 僅依賴 Foundational，可獨立完成與驗證。  
- **User Story 2 (P2)**: 依賴 US1（因為必須已有 org 與 owner），但測試可在獨立環境中驗證。  
- **User Story 3 (P3)**: 依賴 Foundational，並建議在 US1/US2 的基本情境可用後再完成，使測試例更貼近真實使用。  

### Within Each User Story

- 測試（單元／契約／整合） MUST 在實作前撰寫，並先看到失敗再開始實作。  
- Domain 定義與 repository 先於 service / provider，  
- Service / provider 先於 HTTP handler 與路由，  
- 確認該 User Story 的所有測試通過後再移至下一故事。  

### Parallel Opportunities

- Setup 與 Foundational 階段中標記為 [P] 的任務可以由不同開發者平行處理（例如目錄結構、Mongo/Redis 骨架、HTTP router 骨架）。  
- 各 User Story 中標記 [P] 的測試任務（尤其單元與契約測試）可以平行開發。  
- 不同 User Story 在 Foundational 完成後，可以由不同人員並行實作，只需注意共享檔案的修改衝突。  

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. 完成 Phase 1: Setup。  
2. 完成 Phase 2: Foundational（關鍵，阻擋所有 User Story）。  
3. 完成 Phase 3: User Story 1（建立 org + owner）。  
4. **STOP and VALIDATE**：執行所有 US1 測試與基本手動驗證。  
5. 若滿意，部署或在團隊內 demo 作為第一版 MVP。  

### Incremental Delivery

1. 完成 Setup + Foundational → 基礎就緒。  
2. 增量加入 US1 → 測試通過 → 部署 / Demo（MVP）。  
3. 增量加入 US2 → 測試通過 → 部署 / Demo（更細緻權限控制）。  
4. 增量加入 US3 → 測試通過 → 部署 / Demo（抽象 RBAC 介面與可替換實作）。  

### Parallel Team Strategy

在多人團隊情境下：  

1. 全隊先共同完成 Setup + Foundational。  
2. Foundational 完成後：  
   - 開發者 A：專注 US1（org + owner）。  
   - 開發者 B：專注 US2（角色與 CRUD 權限）。  
   - 開發者 C：專注 US3（RBAC 抽象介面與 provider）。  
3. 透過共用測試與契約，確保各故事可以獨立完成與測試，最後再整合。  

---


