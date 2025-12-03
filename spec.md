# Feature Specification: 基本 RBAC 權限系統

**Feature Branch**: `[001-basic-rbac]`  
**Created**: 2025-12-03  
**Status**: Draft  
**Input**: User description: "/speckit.specify 
建立一個基本的rbac, 有role / permission / user / org , role至少有三種 owner admin user , permission至少有crud四種, 將來可以擴充, 需要有一個抽象層, 將來可以替換其他的rbac系統, role不需要繼承"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - 建立組織並指派 Owner (Priority: P1)

一個系統管理者可以透過後台建立一個新的 `org`（組織），並指定一位現有或新建立的使用者為該組織的 `owner` 角色。建立完成後，系統需要確保：  
- 只有 `owner` 可以修改該組織的基本資訊（名稱、描述），  
- 之後的角色指派與權限設定，都會鎖在該 `org` 範圍內。

**Why this priority**: 所有 RBAC 邏輯都以 org 為邊界；沒有 org 與 owner，其餘行為無從談起，是最小 MVP 的必要前置。  

**Independent Test**:  
- 測試流程可以僅啟用「建立 org + 指派 owner」的 API／服務：  
  - 呼叫「建立 org」動作後，檢查資料層有正確的 org 與 user-role 關聯。  
  - 以 `owner` 身分呼叫更新 org API 應成功，以其他角色或匿名呼叫應被拒絕。  
- 不需要啟用其他角色（admin/user）的邏輯也能驗證這一故事。  

**Acceptance Scenarios**:

1. **Given** 系統中存在一個使用者 U，**When** 管理者為 U 建立新 org O 並指派為 owner，**Then** U 對 O 具有 owner 角色且可以修改 O 的基本資訊。  
2. **Given** org O 有一位 owner U1 和另一位一般使用者 U2，**When** U2 嘗試修改 O 的基本資訊，**Then** 請求被拒絕並回傳「權限不足」錯誤。  

---

### User Story 2 - 在單一 org 中管理角色與 CRUD 權限 (Priority: P2)

作為某個 org 的 `owner`，我可以在該 org 範圍內：  
- 將現有使用者指派為 `admin` 或 `user` 角色；  
- 針對特定資源（例如 `project` 或 `record` 類型）設定 CRUD 四種基本 permission（create/read/update/delete）；  
- 系統依據 role 與對應 permission 決定是否允許某個 user 在該 org 內執行對應操作。  

**Why this priority**: 這是 RBAC 的主要價值：在同一 org 內，能依角色精細控制 CRUD 權限，讓 owner 能將管理工作委派給 admin。  

**Independent Test**:  
- 在測試中建立一個 org O、一組使用者（owner O1、admin A1、user U1），  
- 定義一個資源類型（例如 `project`）以及對應 CRUD permission，  
- 驗證：  
  - 有被指派 `admin` 的使用者可以對指定資源做 create/update，  
  - 只有擁有對應 permission 的角色可以執行某動作，其他角色或未登入都會被拒絕。  
- 不需要多 org／跨系統整合也可以完整驗證。  

**Acceptance Scenarios**:

1. **Given** org O 中有 owner O1、admin A1、user U1，且 owner 為 `project` 型資源設定 admin 具備 create/update 權限，**When** A1 嘗試建立新 project，**Then** 操作成功且記錄為由 A1 建立。  
2. **Given** org O 的 `project` 只開放 owner/admin 可以 delete，**When** user 角色 U1 嘗試刪除任一 project，**Then** 請求被拒絕並回傳「權限不足」。  

---

### User Story 3 - 透過抽象 RBAC 介面進行權限查詢 (Priority: P3)

作為應用程式的業務服務（例如某個 `ProjectService`），我希望只依賴一個抽象的 RBAC 介面（例如 `RbacService` 或 `RbacProvider`），  
透過該介面查詢「某 user 是否在某 org 具備某資源與動作的權限」，而不用關心底層是本地實作還是其他 RBAC 系統。  

**Why this priority**: 透過抽象層 decouple 應用程式與具體 RBAC 實作，未來才有空間替換成外部權限系統（例如自建微服務或第三方服務），符合「可擴充且避免過度綁死實作」的要求。  

**Independent Test**:  
- 以單元測試針對抽象介面的本地實作（例如 `InMemoryRbacProvider` 或 `DefaultRbacProvider`）：  
  - 預先塞入 role/permission/user/org 關聯資料，  
  - 呼叫 `can(userId, orgId, resource, action)` 或類似介面，  
  - 驗證回傳結果是否符合預期。  
- 測試不需要依賴實際的 HTTP / DB，只要能替換底層 provider 即可。  

**Acceptance Scenarios**:

1. **Given** RBAC 抽象介面底下有一個本地實作，並已設定 admin 在 org O 上對 `project` 擁有 update 權限，**When** 應用程式呼叫 `rbac.can(adminId, O, "project", "update")`，**Then** 回傳 `true`。  
2. **Given** 對於同樣 org O 與資源型別，user 角色沒有 delete 權限，**When** 應用程式呼叫 `rbac.can(userId, O, "project", "delete")`，**Then** 回傳 `false`，且呼叫過程不需知道底層實作細節。  

---

### Edge Cases

- 當同一 user 屬於多個 org 時，權限判斷必須嚴格以 org 為邊界，禁止「跨 org」竄改資料。  
- 當某 user 在同一 org 同時被指派多個角色（例如 admin + user）時，權限採「聯集」處理（只要任一角色允許即可通過）。  
- 當 role / permission 的設定被移除或變更時，原本允許的操作必須立即反映在權限查詢結果中（避免快取過期問題造成安全洞）。  
- 當抽象 RBAC 介面的底層實作不可用（例如外部權限服務掛掉）時，預設策略應偏向保守（fail closed），避免誤放權限。  

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: 系統 MUST 支援管理 `user`、`org`、`role`、`permission` 四種基本實體，並建立彼此關聯。  
- **FR-002**: 系統 MUST 提供 API 或服務用於建立 `org`，並將指定 `user` 設為該 org 的 `owner` 角色。  
- **FR-003**: 系統 MUST 在每個 org 內至少支援三種角色：`owner`、`admin`、`user`，且 role 不需支援繼承。  
- **FR-004**: 系統 MUST 在每個資源型別上至少支援 CRUD 四種 permission（create/read/update/delete），並可於未來擴充更多動作。  
- **FR-005**: 系統 MUST 能將角色與 permission 綁定（role-permission mapping），並依據 user 在特定 org 的角色集合來判斷實際權限。  
- **FR-006**: 系統 MUST 提供一個抽象的 RBAC 介面（例如 `RbacProvider`），統一對外暴露權限查詢與檢查功能。  
- **FR-007**: 應用程式的業務邏輯 MUST 只依賴 RBAC 抽象介面，而不直接操作底層儲存或第三方 RBAC 實作。  
- **FR-008**: 系統 MUST 支援在不改變上層呼叫端的前提下，替換 RBAC 抽象介面的底層實作（例如從 in-memory 換成資料庫或外部服務）。  
- **FR-009**: 權限檢查 MUST 以 `(user, org, resource, action)` 為最小判斷單位，且任何維度不符時都必須拒絕請求。  
- **FR-010**: 系統 MUST 提供最少一組查詢介面，用於列出某 user 在特定 org 內的所有角色與有效 permission，協助除錯與審計。  

*NEEDS CLARIFICATION（後續可在 plan 中補充）*：  

- **FR-011**: 認證（authentication）機制由哪一層與哪個元件負責？RBAC 僅處理「已識別 user」的授權（authorization）邏輯。  

### Key Entities *(include if feature involves data)*

- **User**: 代表一個登入使用者，具備唯一識別（例如 `user_id`）、基本資訊（email、名稱等），可屬於多個 org。  
- **Org**: 組織邊界，所有 RBAC 判斷都在 org 範圍內進行；包含名稱、描述與建立者等欄位。  
- **Role**: 定義在單一 org 內的角色（owner/admin/user），不支援繼承；一個 user 在同一 org 可有多個角色。  
- **Permission**: 定義對某資源類型與動作的權限，例如 `(resource_type="project", action="create")`。  
- **RolePermission**: 關聯 role 與 permission 的對照表，用於表示某角色在某 org 的允許操作集合。  
- **UserRole**: 關聯 user 與 role 的對照表，通常還會綁定 org，以表示 user 在特定 org 內的角色指派。  

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 針對 User Story 1–3，分別具備獨立自動化測試（單元或整合），且在只啟用對應故事相關路徑時即可通過。  
- **SC-002**: 對於同一組測試資料，使用本地 RBAC 實作與替換成另一種實作（例如 mock provider）時，權限判斷結果一致。  
- **SC-003**: 在單一 org 中，變更某角色的 permission 映射後，對應權限檢查結果在合理時間內（例如 <1 秒）反映變更，無觀察到「舊權限殘留」的情況。  
- **SC-004**: 日常操作下（1000 次權限檢查請求），系統在預期的環境中可於可接受的延遲內完成（具體數字可以在 plan.md 中依實際堆疊補充），且無錯誤率異常升高。  

## Clarifications

### Session 2025-12-03

- Q: 是否需要支援跨 org 的全域管理者角色，或現階段僅支援 org 內部角色？ → A: 僅支援 org 內部角色（owner/admin/user），不實作跨 org 全域管理者；未來有需要再擴充。  


