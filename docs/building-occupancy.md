# Building → Floor → Room Occupancy Specification

**Status:** Active & Locked ✅  
**Upstream:** [docs/business-rules.md](business-rules.md) (`BR-05`, `BR-08`, `BR-17`), [docs/02-functional-requirements.md](02-functional-requirements.md) (`FR-119`–`FR-122`), [database-schema.md](../database-schema.md)  
**Downstream:** [docs/frontend-navigation.md](frontend-navigation.md) (UI presentation & responsive rendering), Stage 05 (Database Design), Stage 06 (API Design)  
**Single Source of Truth:** This document is the **sole authoritative reference** for all domain business logic, active tenant assignment constraints, room/floor/building occupancy status calculations (`OCCUPIED` vs `VACANT`), metric aggregations, and historical isolation rules across the application.

---

## 1. Executive Summary & Core Business Principles

1. **Active Tenant Source of Truth**: Occupancy status across all structural levels (Room, Floor, Building) is derived **exclusively** from currently active tenant assignments (`Tenant.status = ACTIVE` AND `Tenant.currentRoomId = room.id`).
2. **Historical Data Isolation Invariant**: Historical tenant records (`TenancyHistory`) capturing past move-ins and move-outs must be strictly preserved for auditing, room history, and financial billing history, but **MUST NOT** influence current occupancy status. A room whose past tenants have all moved out (`status = MOVED_OUT`) and currently has 0 active assigned tenants is strictly `VACANT`.
3. **Backend Source of Truth**: The NestJS backend database layer is the single authority for occupancy calculations. Frontend components MUST NOT implement independent business rules or override occupancy calculations.
4. **Universal Scalability ($N$-Tier Architecture)**: Occupancy logic and aggregation formulas operate uniformly regardless of system scale ($N$ buildings, $N$ floors per building, $N$ rooms per floor, $N$ tenants per room).

---

## 2. Occupancy Status Calculation Formulas & Rules

### 2.1 Room Occupancy Rules
Each room maintains and exposes an active occupancy status:
- **`OCCUPIED`**: At least one (`≥ 1`) active tenant (`Tenant.status = ACTIVE`) is currently assigned to the room (`Tenant.currentRoomId = Room.id`).
- **`VACANT`**: Zero (`0`) active tenants are currently assigned to the room.

```math
\text{Room Occupancy Status} = \begin{cases} \mathbf{OCCUPIED}, & \text{if } \text{ActiveTenantsCount}(\text{roomId}) \ge 1 \\ \mathbf{VACANT}, & \text{if } \text{ActiveTenantsCount}(\text{roomId}) = 0 \end{cases}
```

#### Operational Examples:
- **Room 01** $\rightarrow$ 2 active tenants assigned $\rightarrow$ **`OCCUPIED`**
- **Room 02** $\rightarrow$ 1 active tenant assigned $\rightarrow$ **`OCCUPIED`**
- **Room 03** $\rightarrow$ 0 active tenants assigned $\rightarrow$ **`VACANT`**
- **Room 04** $\rightarrow$ 1 historical tenant (moved out in July), 0 current active tenants $\rightarrow$ **`VACANT`**

---

### 2.2 Floor Occupancy Rules & Metric Aggregation
A floor aggregates occupancy from all child rooms belonging to that floor (`Room.floorId = Floor.id`):
- **`OCCUPIED`**: At least one (`≥ 1`) room on that floor has a status of `OCCUPIED`.
- **`VACANT`**: ALL rooms on that floor have a status of `VACANT` (or 0 active tenants exist on the floor).

```math
\text{Floor Occupancy Status} = \begin{cases} \mathbf{OCCUPIED}, & \text{if } \exists \, \text{room} \in \text{FloorRooms} \text{ where } \text{RoomStatus} = \mathbf{OCCUPIED} \\ \mathbf{VACANT}, & \text{if } \forall \, \text{room} \in \text{FloorRooms}, \text{RoomStatus} = \mathbf{VACANT} \end{cases}
```

#### Floor Aggregated Metrics
Every floor domain model MUST compute and expose:
1. **`totalRooms`**: Count of all registered rooms on the floor ($N_{\text{rooms}}$).
2. **`occupiedRooms`**: Count of rooms on the floor where $\text{ActiveTenants} \ge 1$.
3. **`vacantRooms`**: Count of rooms on the floor where $\text{ActiveTenants} = 0$.
4. **`totalActiveTenants`**: Sum of active tenants across all rooms on the floor ($\sum \text{ActiveTenants}$).

#### Operational Examples:
- **Floor 1**:
  - Room 11 $\rightarrow$ 2 active tenants (**Occupied**)
  - Room 12 $\rightarrow$ 0 active tenants (**Vacant**)
  - Room 13 $\rightarrow$ 1 active tenant (**Occupied**)  
  $\rightarrow$ **Floor 1 Status:** **`OCCUPIED`** (`totalRooms: 3`, `occupiedRooms: 2`, `vacantRooms: 1`, `totalActiveTenants: 3`)

- **Floor 2**:
  - Room 21 $\rightarrow$ 0 active tenants (**Vacant**)
  - Room 22 $\rightarrow$ 0 active tenants (**Vacant**)  
  $\rightarrow$ **Floor 2 Status:** **`VACANT`** (`totalRooms: 2`, `occupiedRooms: 0`, `vacantRooms: 2`, `totalActiveTenants: 0`)

---

### 2.3 Building Occupancy Rules & Metric Aggregation
A building aggregates occupancy from all child floors and rooms (`Floor.buildingId = Building.id`):
- **`OCCUPIED`**: At least one (`≥ 1`) room across any floor in the building is `OCCUPIED`.
- **`VACANT`**: ALL rooms across ALL floors in the building are `VACANT`.

#### Building Aggregated Metrics
Every building domain model MUST compute and expose:
1. **`totalFloors`**: Total number of registered floors ($N_{\text{floors}}$).
2. **`totalRooms`**: Total number of registered rooms across all floors ($N_{\text{rooms}}$).
3. **`occupiedRooms`**: Total count of occupied rooms in the building.
4. **`vacantRooms`**: Total count of vacant rooms in the building.
5. **`occupiedFloors`**: Count of floors with status `OCCUPIED`.
6. **`vacantFloors`**: Count of floors with status `VACANT`.
7. **`totalActiveTenants`**: Total count of active tenants living in the building.
8. **`overallOccupancyStatus`**: Overall status (`OCCUPIED` | `VACANT`).

---

## 3. Backend Source of Truth & Synchronization Invariants

To eliminate state drift and inconsistent cached values across multi-user sessions, occupancy states are governed by rigid backend invariants:

1. **Dynamic Derivation Strategy**:
   - Occupancy status and metrics are computed dynamically via SQL/Prisma relational aggregations during query execution to guarantee real-time accuracy.
   - Database SQL Pattern:
     ```sql
     SELECT 
       r.id AS room_id,
       COUNT(t.id) FILTER (WHERE t.status = 'ACTIVE') AS active_tenant_count,
       CASE WHEN COUNT(t.id) FILTER (WHERE t.status = 'ACTIVE') > 0 THEN 'OCCUPIED' ELSE 'VACANT' END AS occupancy_status
     FROM rooms r
     LEFT JOIN tenants t ON t.current_room_id = r.id
     WHERE r.floor_id = $1
     GROUP BY r.id;
     ```
2. **Database Indexing for Performance**:
   - Composite index `@@index([currentRoomId, status])` on the `tenants` table enables sub-millisecond aggregation ($< 5\text{ ms}$) even across 10,000+ tenant records.
3. **Automatic Event Synchronization**:
   - Any mutation affecting tenant-room mapping immediately updates query results without background jobs or manual refresh:
     - Tenant onboarding / room assignment ($\text{ACTIVE}$ assigned $\rightarrow$ Room transitions to `OCCUPIED`)
     - Tenant room transfer (Old room re-evaluated; new room transitions to `OCCUPIED`)
     - Tenant checkout / move-out ($\text{status} \rightarrow \text{MOVED\_OUT}$, `currentRoomId` set to `NULL` $\rightarrow$ Room re-evaluated)
     - Tenant deactivation ($\text{status} \rightarrow \text{INACTIVE}$)
     - New room creation (Initial status defaults to `VACANT`)
     - Room removal / deactivation (Removed from floor/building aggregates)

---

## 4. Scalability Architecture ($N$-Tier Support)

The occupancy calculation system is engineered to scale seamlessly:
- **Scalability Target**: Supports $N$ buildings, $N$ floors per building, $N$ rooms per floor, and $N$ tenants per room without code or schema alterations.
- **Database Optimization**: All hierarchical foreign keys (`Building.landlordId`, `Floor.buildingId`, `Room.floorId`, `Tenant.currentRoomId`) and status fields are indexed.
- **API Response SLA**: Building/Floor occupancy metrics respond in $\le 100\text{ ms}$ (p95) for buildings with up to 50 floors and 500 rooms.

---

## 5. Maintenance Policy & Future Occupancy Rules

Whenever new business rules regarding occupancy calculations, tenant assignment invariants, or structural aggregation metrics are introduced in future stages:
1. **Update ONLY this document** (`building-occupancy.md`) with the authoritative logic.
2. Cross-reference this document from `docs/business-rules.md` (`BR-17`), `docs/02-functional-requirements.md`, and `docs/frontend-navigation.md`.
3. **Do NOT duplicate detailed calculation formulas** across other documents.
