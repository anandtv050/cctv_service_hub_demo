# ServiceHub — Task / Project Workflow (developer reference)

End-to-end lifecycle of a ticket, from the customer's first call to closure.
This is the internal logic spec — **not** an end-user guide. Diagrams use
Mermaid (renders in GitHub / VS Code Markdown preview).

---

## 1. Actors

| Actor | Role in the flow |
|---|---|
| **Customer** | Raises the request (missed call → SMS link → form). Sees only their own ticket. |
| **Admin** | Org-wide. Triages, classifies, assigns; relocates technicians across cost centers; sets availability/capacity. |
| **PM (Project Manager)** | Owns a cost center's work. Assigns to technicians, **approves/rejects** completed work. |
| **Technician** | Field worker. Executes the task, submits the **Daily Activity Report (DAR)** + photos. |
| **System** | Auto-creates ticket from missed call, runs SLA timers, sends notifications. |

---

## 2. Ticket model (the unit of work)

A **ticket = task**. A "project" is just a ticket with `type = Project`.

| Field | Values / notes |
|---|---|
| `id` | `SR-####` (Service), `PR-####` (Project), `OT-####` (Other) |
| `type` | **Service** \| **Project** \| **Other** |
| `priority` | Urgent \| High \| Medium \| Low |
| `cost_center` | Calicut \| Kollam \| Ernakulam |
| `customer` | name / site |
| `assignee` | technician (nullable until assigned) |
| `status` | New → Assigned → In Progress → With PM → Done (+ Overdue flag) |
| `dar` | Daily Activity Report (required to close) + photos |

---

## 3. Status lifecycle (state machine)

```mermaid
stateDiagram-v2
    [*] --> New: customer call → ticket created
    New --> Assigned: Admin/PM classifies + assigns technician
    Assigned --> InProgress: technician taps "Start"
    InProgress --> WithPM: technician submits DAR + photos
    WithPM --> Done: PM approves
    WithPM --> InProgress: PM requests changes (rework)
    Done --> [*]

    Assigned --> Overdue: SLA breached
    InProgress --> Overdue: SLA breached
    Overdue --> WithPM: technician submits DAR
    Overdue --> InProgress: reassigned / resumed

    note right of WithPM
        DAR is mandatory: all fields + ≥1 site photo.
        This is the gate between "work done" and "closed".
    end note
```

**Status meanings**

| Status | Meaning | Who acts next |
|---|---|---|
| **New** | Created, unassigned | Admin / PM |
| **Assigned** | Given to a technician (shows in their **To-do**) | Technician |
| **In Progress** | Technician started on site | Technician |
| **With PM** | DAR submitted, awaiting sign-off | PM |
| **Done** | PM approved → closed (billable) | — |
| **Overdue** | SLA breached (overlay flag, not a dead end) | Admin / PM |

---

## 4. End-to-end flow (call → close)

```mermaid
flowchart TD
    A([Customer gives missed call]) --> B[System: auto-send SMS with form link]
    B --> C[Customer fills service form]
    C --> D[/Ticket created · status = New/]

    D --> E{Admin/PM triage}
    E --> F[Classify type: Service / Project / Other]
    F --> G[Set priority + cost center]
    G --> H[Assign technician · status = Assigned]

    H --> I[Technician sees it in To-do]
    I --> J[Tap Start · status = In Progress]
    J --> K[Do the work on site]
    K --> L[Fill Daily Activity Report\nall fields + site photo]
    L --> M[/Submit · status = With PM/]

    M --> N{PM reviews in Approvals}
    N -->|Approve| O([status = Done · ticket closed])
    N -->|Request changes| J

    %% admin override lane
    H -.reassign / relocate tech.-> H
    M -.admin can reassign.-> H

    O --> P[Feeds reports: SLA, CSAT, leaderboard]
```

---

## 5. Who-does-what (sequence view)

```mermaid
sequenceDiagram
    participant C as Customer
    participant S as System
    participant A as Admin/PM
    participant T as Technician
    participant P as PM

    C->>S: Missed call
    S-->>C: SMS with form link
    C->>S: Submit form
    S->>A: New ticket (status: New)
    A->>A: Classify (type/priority/cost center)
    A->>T: Assign (status: Assigned)
    T->>T: Start (status: In Progress)
    T->>T: Do work
    T->>P: Submit DAR + photos (status: With PM)
    alt Work OK
        P->>S: Approve (status: Done)
        S-->>C: Closure notification
    else Needs rework
        P->>T: Request changes (status: In Progress)
    end
```

---

## 6. Service vs Project — same flow, different shape

| | **Service** ticket | **Project** ticket |
|---|---|---|
| Scope | one visit, one job | multi-visit, multi-day |
| DAR | one DAR closes it | a DAR per visit; progress % tracked |
| Closure | single PM approval → Done | progress reaches 100% + final approval |
| Example | "AC not cooling" | "Hotel — 12-camera CCTV install" |

> Same state machine applies to both. A Project simply loops
> `In Progress → With PM → (approve) → In Progress` across visits until
> `progress = 100%`, then the final approval sets it to **Done**.

---

## 7. Admin overrides (cross-cutting)

These don't change the ticket states, but the Admin can act at any point:

```mermaid
flowchart LR
    subgraph Admin powers
      R1[Reassign ticket to another technician]
      R2[Relocate technician across cost centers]
      R3[Set technician availability: Active / On Leave]
      R4[Adjust technician capacity]
    end
    R2 --> LB[Rebalances cost-center load]
    R3 --> LB
    R4 --> LB
```

- **Reassign** → ticket's `assignee` changes (stays in same status).
- **Relocate / availability / capacity** → affects *future* assignment & the
  Deployment load board; does not auto-move existing tickets.

---

## 8. Notifications (triggers)

| Event | Notify | Channel |
|---|---|---|
| Ticket created | Admin/PM | in-app / email |
| Ticket assigned | Technician | push (mobile) |
| DAR submitted | PM | in-app / push |
| Approved / closed | Customer + Technician | SMS / push |
| Changes requested | Technician | push |
| SLA breach (Overdue) | PM + Admin | in-app / email |

---

## 9. Implementation notes (for the real build)

- **State transitions** should be enforced server-side (a ticket can't jump
  `New → Done`). Keep a `status_history` / audit table (who, when, from→to).
- **DAR** is the hard gate: server validates required fields + ≥1 photo before
  allowing `In Progress → With PM`.
- **Photos** → object storage (S3 / R2), signed URLs, mobile-side compression.
- **Permissions (RLS):** Customer = own ticket; Technician = own assigned;
  PM = own cost center(s); Admin = all.
- **Overdue** is a derived flag from SLA timer, not a manual status.
