# Workflow — Task (Service) vs Project

Same data model (a ticket), but the **lifecycle differs**. A **task** is one
visit → one report → one approval → done. A **project** is a *container* of
many task-cycles, tracked by progress, ending in a final handover.

Developer reference. Mermaid diagrams render in GitHub / VS Code preview.

---

## 1. The core difference

| | **Task** (Service / Other) | **Project** |
|---|---|---|
| Size | one visit, one job | multi-visit, multi-day |
| Structure | single ticket | **parent that contains child tasks** |
| Technician | usually one | often a team |
| Reports (DAR) | **one** DAR closes it | **one DAR per visit** |
| Approval | single PM approval → Done | approval per visit **+ final handover** |
| Tracking | status only | **progress %** across child tasks |
| Closure | PM approves | progress = 100% **+** final sign-off |
| Example | "AC not cooling" | "Hotel — 12-camera CCTV install" |

> **Data note:** still one `tickets` table. A project is a ticket with
> `type = Project` that has **child tickets** (`parent_id`). Each child is an
> ordinary task with its own DAR cycle. Project progress = % of children Done.

---

## 2. TASK flow (Service ticket) — simple, linear

```mermaid
flowchart TD
    TA([Ticket created · New]) --> TB[Admin/PM assigns technician · Assigned]
    TB --> TC[Technician taps Start · In Progress]
    TC --> TD[Do work on site]
    TD --> TE[Submit DAR + photo · With PM]
    TE --> TF{PM reviews}
    TF -->|Approve| TG([Done · closed])
    TF -->|Request changes| TC
```

**One cycle. One DAR. One approval. Done.**

```mermaid
stateDiagram-v2
    [*] --> New
    New --> Assigned
    Assigned --> InProgress
    InProgress --> WithPM: submit DAR
    WithPM --> Done: approve
    WithPM --> InProgress: reject
    Done --> [*]
```

---

## 3. PROJECT flow — a container of task-cycles

A project adds **planning** up front and a **final handover** at the end, and
loops through its child tasks/visits in the middle.

```mermaid
flowchart TD
    PA([Project request · New]) --> PB[PM/Admin: site survey + scope]
    PB --> PC[Break into child tasks / visits + milestones]
    PC --> PD[Assign team + schedule · In Progress]

    PD --> LOOP{For each child task / visit}
    LOOP --> PF[Technician: Start → work → DAR + photos]
    PF --> PG{PM approves this visit?}
    PG -->|Request changes| PF
    PG -->|Approve| PH[Mark child done · update progress %]
    PH --> PI{All children done? progress = 100%?}
    PI -->|No| LOOP
    PI -->|Yes| PJ[Final review / handover / customer sign-off]
    PJ --> PK([Project Closed · Done])
```

**Many cycles. Many DARs. Progress accrues. Final handover closes it.**

```mermaid
stateDiagram-v2
    [*] --> New
    New --> Planning: survey + scope
    Planning --> InProgress: tasks created + assigned
    InProgress --> InProgress: child task approved (progress %↑)
    InProgress --> Review: progress = 100%
    Review --> InProgress: handover rejected / punch-list
    Review --> Done: final sign-off
    Done --> [*]
```

---

## 4. How a project contains tasks (hierarchy)

```mermaid
flowchart TD
    P[PROJECT: Hotel Sea View — 12-camera CCTV\nprogress 0% → 100%]
    P --> T1[Task: Site survey]
    P --> T2[Task: Mount cameras — Floor 1]
    P --> T3[Task: Mount cameras — Floor 2]
    P --> T4[Task: NVR install + cabling]
    P --> T5[Task: Testing + handover]

    T1 -. each child runs the TASK flow .-> X[(Assign → In Progress →\nDAR → Approve → Done)]
```

Every child task in §4 runs the **Task flow** from §2. When the last child is
approved, the parent project hits 100% and goes to **final handover**.

---

## 5. Side-by-side timeline

```mermaid
flowchart LR
    subgraph TASK
      direction LR
      a1[Assign] --> a2[Work] --> a3[DAR] --> a4[Approve] --> a5[Done]
    end
    subgraph PROJECT
      direction LR
      b1[Plan] --> b2[Visit 1\nDAR+approve] --> b3[Visit 2\nDAR+approve] --> b4[Visit N\nDAR+approve] --> b5[Handover] --> b6[Closed]
    end
```

---

## 6. What changes for the build

| Concern | Task | Project |
|---|---|---|
| Records | 1 ticket | 1 parent + N child tickets (`parent_id`) |
| Progress | derived from status | `done_children / total_children` |
| Approvals | 1 | N (per visit) + 1 final |
| DARs | 1 | many (1 per visit) |
| Extra screens | — | project detail with child-task list + progress, handover step |
| Permissions | same | PM sees whole project tree; technician sees only their child task |

> **MVP shortcut:** if time is tight, ship Projects as a *flat* ticket with a
> manual `progress %` field (no child tasks). Add the parent→child task tree in
> Phase 2 — the diagrams above are the target model.
