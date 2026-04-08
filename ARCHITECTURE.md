# CAMBUS Operations Platform - Architecture & Planning

## PHASE 1: PRODUCT & ARCHITECTURE PLAN

### 1.1 High-Level Architecture

**Stack Overview:**
- **Frontend:** Next.js 14+ (App Router) + React 18 + TypeScript
- **Backend:** Next.js API Routes + Server Actions (hybrid approach)
- **Database:** PostgreSQL + Prisma ORM
- **Auth:** NextAuth.js v5 (simple, internal-only, role-based)
- **UI:** Tailwind CSS + shadcn/ui
- **State Management:** React Query (TanStack Query) for server state, React Context for auth/UI state
- **Validation:** Zod (schema validation)
- **Utilities:** date-fns, papaparse (CSV export)

**Why these choices:**
- Next.js API Routes: Single codebase, type-safe, no separate backend infrastructure
- Server Actions: Reduces API boilerplate, direct DB access from forms
- Prisma: Type-safe DB interactions, migrations built in, excellent DX
- NextAuth: Minimal auth overhead, good RBAC support, built for Next.js
- React Query: Handles server state sync, caching, background refetch automatically
- TailwindCSS + shadcn/ui: Pre-built components = faster UI delivery, consistent

---

### 1.2 MVP Scope (Version 1)

**IN SCOPE:**
- ✅ Employee CRUD + profile pages
- ✅ Qualifications CRUD (attach to employees)
- ✅ Availability management (recurring only, no temporal overrides yet)
- ✅ Basic shift/assignment creation + employee assignment
- ✅ Sign-off/incident logging (attach to employee ± shift)
- ✅ Shift points tracking (manual entry only)
- ✅ Dashboard with key metrics
- ✅ Simple reporting (tables, CSV export)
- ✅ Basic audit trail (created_at, updated_at, updated_by)
- ✅ Admin/Supervisor role-based access

**OUT OF SCOPE (Phase 2+):**
- ❌ Automated conflict detection
- ❌ AI-driven shift assignment suggestions
- ❌ Expiration alerts / email workflows
- ❌ 3-sign-off escalation rules
- ❌ Point threshold actions
- ❌ Temporal availability overrides
- ❌ Mobile app
- ❌ Complex reporting / BI dashboards
- ❌ Real-time collaboration features

**Why this scope:**
- Strong MVP backbone (all main entities + CRUD + dashboarding)
- ~4-6 weeks of solid development
- Rule automation hooks in place but not implemented
- Can demo internally and get feedback before adding complexity

---

### 1.3 Architecture Principles

1. **Modular business logic:** Service layer (`src/lib/services`) separates concerns from API endpoints
2. **Type safety first:** TypeScript throughout, Zod validation on inputs
3. **Permission-aware:** Every API route/action checks user role + permissions
4. **Auditability:** Key mutations logged with user reference
5. **Rule hooks:** Business rules (point thresholds, expiration, escalation) as pluggable services
6. **Clean data flow:** API routes → Services → Prisma → DB (unidirectional)
7. **Consistent error handling:** Standardized error responses across all endpoints
8. **Form-first UX:** Server actions for form submissions, React Query for queries

---

### 1.4 Authentication Model

**Users Table:**
- `id, email, password_hash, role, name, active, created_at`
- **Roles:** `ADMIN` (full access), `SUPERVISOR` (most features, no user mgmt)
- Session-based via NextAuth (JWT optional)
- Protected API routes: middleware checks session → checks role

**Future expansion:** Dispatcher, DataAssistant roles (same table, different permissions)

---

## PHASE 2: PRISMA SCHEMA & DATA MODEL

### 2.1 Entity Relationships Diagram

```
User (1) → (many) AuditLog
        → (many) EmployeeNote
        → (many) SignOff

Employee (1) → (many) Qualification (join)
            → (many) Availability
            → (many) ShiftAssignment
            → (many) SignOff
            → (many) ShiftPoint
            → (many) SpecialServiceStaff
            → (many) EmployeeNote
            → (many) AuditLog

Qualification (many) → (1) Employee

Availability (many) → (1) Employee

Shift (1) → (many) ShiftAssignment
         → (many) SignOff (optional)
         → (many) RequiredQualification

ShiftAssignment (many) → (1) Employee
                      → (1) Shift

SignOff (many) → (1) Employee
              → (1) Shift (optional)
              → (1) User (entered_by)

ShiftPoint (many) → (1) Employee
                 → (1) User (entered_by)

SpecialService (1) → (many) SpecialServiceStaff
                  → (many) SpecialServiceNote

SpecialServiceStaff (many) → (1) Employee
                          → (1) SpecialService

EmployeeNote (many) → (1) Employee
                   → (1) User (entered_by)

AuditLog (many) → (1) User (performed_by)
```

### 2.2 Full Prisma Schema

```prisma
// User / Auth
model User {
  id        String    @id @default(cuid())
  email     String    @unique
  name      String
  role      Role      @default(SUPERVISOR)
  active    Boolean   @default(true)
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt

  auditLogs      AuditLog[]
  employeeNotes  EmployeeNote[]
  signOffs       SignOff[]
  shiftPoints    ShiftPoint[]

  @@map("users")
}

enum Role {
  ADMIN
  SUPERVISOR
}

// Employee Core
model Employee {
  id              String    @id @default(cuid())
  universityEmail String    @unique
  name            String
  phone           String?
  employeeId      String    @unique // university ID
  hireDate        DateTime
  status          EmployeeStatus @default(ACTIVE)
  notes           String?
  
  // Qualifications (many-to-many via join table)
  qualifications  EmployeeQualification[]
  
  // Related data
  availability    Availability[]
  shiftAssignments ShiftAssignment[]
  signOffs        SignOff[]
  shiftPoints     ShiftPoint[]
  specialServiceStaff SpecialServiceStaff[]
  employeeNotes   EmployeeNote[]
  
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  @@index([status])
  @@index([universityEmail])
  @@map("employees")
}

enum EmployeeStatus {
  ACTIVE
  INACTIVE
  ON_LEAVE
}

// Qualifications
model Qualification {
  id          String    @id @default(cuid())
  name        String    @unique // e.g., "Dispatch Certified", "Bionic Specialist"
  description String?
  
  employees   EmployeeQualification[]
  shiftRequirements ShiftQualificationRequirement[]
  
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt

  @@map("qualifications")
}

// Join table: Employee → Qualifications
model EmployeeQualification {
  id            String    @id @default(cuid())
  employee      Employee  @relation(fields: [employeeId], references: [id], onDelete: Cascade)
  employeeId    String
  qualification Qualification @relation(fields: [qualificationId], references: [id], onDelete: Cascade)
  qualificationId String
  
  dateEarned   DateTime
  expiresAt    DateTime?
  notes        String?
  
  createdAt    DateTime  @default(now())
  updatedAt    DateTime  @updatedAt

  @@unique([employeeId, qualificationId])
  @@index([expiresAt])
  @@map("employee_qualifications")
}

// Availability
model Availability {
  id            String    @id @default(cuid())
  employee      Employee  @relation(fields: [employeeId], references: [id], onDelete: Cascade)
  employeeId    String
  
  dayOfWeek     Int       // 0=Sunday, 6=Saturday
  startTime     String    // "08:00" format
  endTime       String    // "17:00" format
  
  availabilityType AvailabilityType @default(RECURRING)
  validFrom     DateTime?
  validTo       DateTime?
  
  restrictions  String?   // e.g., "No split shifts", "Prefer afternoon"
  
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt

  @@index([employeeId])
  @@map("availability")
}

enum AvailabilityType {
  RECURRING
  TEMPORARY
}

// Shifts / Assignments
model Shift {
  id              String    @id @default(cuid())
  shiftType       ShiftType
  routeOrService  String    // "Route 5", "Special Event Service"
  scheduledDate   DateTime
  startTime       String    // "08:00"
  endTime         String    // "17:00"
  
  staffingNeeded  Int       @default(1)
  requiredQualifications ShiftQualificationRequirement[]
  
  assignments     ShiftAssignment[]
  signOffs        SignOff[]   // sign-offs linked to this shift
  
  notes           String?
  
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  @@index([scheduledDate])
  @@index([shiftType])
  @@map("shifts")
}

enum ShiftType {
  ROUTE
  SPECIAL_SERVICE
  MAINTENANCE
  TRAINING
}

// Shift → Qualifications (requirements)
model ShiftQualificationRequirement {
  id              String    @id @default(cuid())
  shift           Shift     @relation(fields: [shiftId], references: [id], onDelete: Cascade)
  shiftId         String
  qualification   Qualification @relation(fields: [qualificationId], references: [id], onDelete: Cascade)
  qualificationId String

  @@unique([shiftId, qualificationId])
  @@map("shift_qualification_requirements")
}

// Shift Assignments (Employee assigned to Shift)
model ShiftAssignment {
  id          String    @id @default(cuid())
  shift       Shift     @relation(fields: [shiftId], references: [id], onDelete: Cascade)
  shiftId     String
  employee    Employee  @relation(fields: [employeeId], references: [id], onDelete: Cascade)
  employeeId  String
  
  assignmentStatus AssignmentStatus @default(ASSIGNED)
  
  notes       String?
  
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt

  @@unique([shiftId, employeeId])
  @@index([assignmentStatus])
  @@map("shift_assignments")
}

enum AssignmentStatus {
  ASSIGNED
  CONFIRMED
  COMPLETED
  CANCELLED
}

// Sign-offs / Incidents
model SignOff {
  id          String    @id @default(cuid())
  employee    Employee  @relation(fields: [employeeId], references: [id], onDelete: Cascade)
  employeeId  String
  
  shift       Shift?    @relation(fields: [shiftId], references: [id], onDelete: SetNull)
  shiftId     String?
  
  signOffType SignOffType
  reason      String
  
  enteredBy   User      @relation(fields: [enteredById], references: [id])
  enteredById String
  
  notes       String?
  resolved    Boolean   @default(false)
  
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt

  @@index([employeeId])
  @@index([createdAt])
  @@index([resolved])
  @@map("sign_offs")
}

enum SignOffType {
  ABSENCE
  TARDY
  EARLY_DEPARTURE
  INCIDENT
  NOTE
}

// Shift Points / Performance
model ShiftPoint {
  id          String    @id @default(cuid())
  employee    Employee  @relation(fields: [employeeId], references: [id], onDelete: Cascade)
  employeeId  String
  
  eventSource String    // e.g., "absence", "incident", "commendation"
  pointValue  Int       // positive or negative
  pointCategory PointCategory
  
  applicableDate DateTime
  
  enteredBy   User      @relation(fields: [enteredById], references: [id])
  enteredById String
  
  notes       String?
  
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt

  @@index([employeeId])
  @@index([applicableDate])
  @@map("shift_points")
}

enum PointCategory {
  ATTENDANCE
  PERFORMANCE
  CONDUCT
  COMMENDATION
}

// Special Services
model SpecialService {
  id              String    @id @default(cuid())
  serviceType     String    // "Game Day", "Campus Event", "Charter"
  serviceDate     DateTime
  semester        String?   // "Spring 2024"
  
  staffingNeeded  Int
  staffFilled     Int       @default(0)
  hoursPerStaff   Float     @default(8.0)
  
  staff           SpecialServiceStaff[]
  notes           SpecialServiceNote[]
  
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  @@index([serviceDate])
  @@index([semester])
  @@map("special_services")
}

// Special Service Staff Assignment
model SpecialServiceStaff {
  id              String    @id @default(cuid())
  service         SpecialService @relation(fields: [serviceId], references: [id], onDelete: Cascade)
  serviceId       String
  employee        Employee  @relation(fields: [employeeId], references: [id], onDelete: Cascade)
  employeeId      String
  
  hoursWorked     Float?
  status          SpecialServiceStatus @default(ASSIGNED)
  
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  @@unique([serviceId, employeeId])
  @@map("special_service_staff")
}

enum SpecialServiceStatus {
  ASSIGNED
  CONFIRMED
  WORKED
  CANCELLED
}

// Employee Notes / Personnel Notes
model EmployeeNote {
  id          String    @id @default(cuid())
  employee    Employee  @relation(fields: [employeeId], references: [id], onDelete: Cascade)
  employeeId  String
  
  noteType    NoteType
  visibility  VisibilityLevel @default(INTERNAL)
  
  content     String
  enteredBy   User      @relation(fields: [enteredById], references: [id])
  enteredById String
  
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt

  @@index([employeeId])
  @@index([createdAt])
  @@map("employee_notes")
}

enum NoteType {
  GENERAL
  PERFORMANCE
  COMMENDATION
  CONCERN
  FOLLOWUP
}

enum VisibilityLevel {
  INTERNAL      // admins only
  SUPERVISOR    // supervisors + admins
  EMPLOYEE      // employee can see
}

// Audit Log
model AuditLog {
  id          String    @id @default(cuid())
  action      String    // "created", "updated", "deleted"
  entityType  String    // "Employee", "Shift", etc.
  entityId    String
  
  performedBy User      @relation(fields: [performedById], references: [id])
  performedById String
  
  changes     String    // JSON stringified: { before: {...}, after: {...} }
  
  createdAt   DateTime  @default(now())

  @@index([entityType, entityId])
  @@index([createdAt])
  @@map("audit_logs")
}
```

### 2.3 Data Model Reasoning

| Entity | Why | Key Design |
|--------|-----|-----------|
| User | Auth + audit trail requirement | Role-based with ADMIN/SUPERVISOR split |
| Employee | Core entity, needs many relationships | Status enum for active/inactive/on-leave |
| Qualification | Many-to-many with Employee (join table) | Supports expiration dates for future alerts |
| EmployeeQualification | Tracks expiry + date earned | Indexed on expiresAt for future queries |
| Availability | Recurring patterns + future overrides | AvailabilityType enum allows extensibility |
| Shift | Core scheduling unit | Links to qualifications & sign-offs |
| ShiftAssignment | Employee → Shift binding | Status tracks lifecycle (assigned→confirmed→completed) |
| SignOff | Incidents/absences (current) + escalation rules (future) | Resolved flag allows workflow |
| ShiftPoint | Performance tracking (manual now, computed later) | Category enum enables future rule triggers |
| SpecialService | Game days, events, charters | Separate from regular shifts for easy filtering |
| SpecialServiceStaff | Many-to-many SpecialService ↔ Employee | Tracks hours + confirmation status |
| EmployeeNote | Personnel records with visibility control | Extensible: performance reviews, concerns, commendations |
| AuditLog | Compliance + debugging | JSON changes column allows flexibility |

---

## PHASE 3: ROUTES, LAYOUTS, AND COMPONENTS

### 3.1 Folder Structure

```
src/
├── app/                           # Next.js App Router
│   ├── (auth)/                    # Auth pages (outside main layout)
│   │   ├── login/page.tsx
│   │   └── layout.tsx
│   ├── (dashboard)/               # Main app (protected layout)
│   │   ├── layout.tsx             # Sidebar + top nav
│   │   ├── page.tsx               # Dashboard
│   │   ├── employees/
│   │   │   ├── page.tsx           # Employee list
│   │   │   ├── [id]/page.tsx      # Employee profile
│   │   │   ├── [id]/edit/page.tsx
│   │   │   └── new/page.tsx
│   │   ├── qualifications/
│   │   │   ├── page.tsx
│   │   │   └── [id]/page.tsx
│   │   ├── availability/
│   │   │   ├── page.tsx
│   │   │   └── [employeeId]/page.tsx
│   │   ├── shifts/
│   │   │   ├── page.tsx
│   │   │   ├── [id]/page.tsx
│   │   │   ├── [id]/edit/page.tsx
│   │   │   └── new/page.tsx
│   │   ├── sign-offs/
│   │   │   ├── page.tsx
│   │   │   ├── [id]/page.tsx
│   │   │   └── new/page.tsx
│   │   ├── points/
│   │   │   └── page.tsx           # Summary + add form
│   │   ├── special-services/
│   │   │   ├── page.tsx
│   │   │   ├── [id]/page.tsx
│   │   │   ├── [id]/edit/page.tsx
│   │   │   └── new/page.tsx
│   │   ├── reporting/
│   │   │   ├── page.tsx           # Reporting hub
│   │   │   ├── employees/page.tsx
│   │   │   └── staffing-gaps/page.tsx
│   │   ├── admin/
│   │   │   ├── users/page.tsx     # User management (ADMIN only)
│   │   │   └── audit/page.tsx     # Audit log viewer
│   │   └── api/                   # API routes (inside dashboard for auth)
│   │       ├── employees/[...route].ts
│   │       ├── qualifications/[...route].ts
│   │       ├── shifts/[...route].ts
│   │       └── ...
│   └── api/                       # Public API (auth routes)
│       ├── auth/[...nextauth].ts
│       └── health/route.ts
│
├── components/
│   ├── layout/
│   │   ├── Sidebar.tsx
│   │   ├── TopNav.tsx
│   │   └── ProtectedLayout.tsx
│   ├── shared/
│   │   ├── DataTable.tsx          # Generic table with filtering
│   │   ├── SearchBar.tsx
│   │   ├── FilterPanel.tsx
│   │   ├── ConfirmDialog.tsx
│   │   ├── Loading.tsx
│   │   └── EmptyState.tsx
│   ├── forms/
│   │   ├── EmployeeForm.tsx       # Create/Edit forms
│   │   ├── QualificationForm.tsx
│   │   ├── ShiftForm.tsx
│   │   ├── SignOffForm.tsx
│   │   ├── AvailabilityForm.tsx
│   │   └── ShiftPointForm.tsx
│   ├── employee/
│   │   ├── EmployeeTable.tsx
│   │   ├── EmployeeProfile.tsx
│   │   ├── QualificationsList.tsx
│   │   ├── AvailabilityView.tsx
│   │   └── SignOffHistory.tsx
│   ├── dashboard/
│   │   ├── MetricCards.tsx
│   │   ├── RecentSignOffs.tsx
│   │   ├── StaffingGaps.tsx
│   │   └── SpecialServicesOverview.tsx
│   └── ui/                        # shadcn/ui components (generated)
│       ├── button.tsx
│       ├── table.tsx
│       ├── dialog.tsx
│       └── ...
│
├── lib/
│   ├── auth.ts                    # NextAuth config
│   ├── db.ts                      # Prisma client
│   ├── constants.ts               # Enums, config constants
│   ├── permissions.ts             # RBAC helpers
│   ├── validators.ts              # Zod schemas
│   ├── services/                  # Business logic layer
│   │   ├── employee.service.ts
│   │   ├── shift.service.ts
│   │   ├── qualification.service.ts
│   │   ├── shift-points.service.ts
│   │   ├── rules.service.ts       # Future: escalation, thresholds
│   │   ├── audit.service.ts
│   │   └── export.service.ts      # CSV export
│   ├── hooks/                     # React hooks
│   │   ├── useAuth.ts
│   │   ├── useQuery.ts            # React Query wrappers
│   │   └── useForm.ts
│   └── utils/
│       ├── api.ts                 # API client helpers
│       ├── dates.ts
│       ├── formatting.ts
│       └── validation.ts
│
├── actions/                       # Server actions (form submissions)
│   ├── employee.actions.ts
│   ├── shift.actions.ts
│   ├── qualification.actions.ts
│   ├── sign-off.actions.ts
│   └── ...
│
├── prisma/
│   ├── schema.prisma              # Data model (see Phase 2)
│   ├── seed.ts                    # Seed data
│   └── migrations/
│
├── middleware.ts                  # NextAuth session check
├── env.local                      # Environment variables (template)
└── tsconfig.json
```

### 3.2 Page Routes (User-Facing)

| Route | Purpose | Features |
|-------|---------|----------|
| `/` (auth'd) | Dashboard | Metrics, recent activity, quick actions |
| `/employees` | List | Table, search, filter by status/role, bulk actions |
| `/employees/new` | Create | Form with all basic fields |
| `/employees/[id]` | Profile | Read-only profile + quick edit buttons |
| `/employees/[id]/edit` | Edit | Full form, audit trail shown |
| `/qualifications` | List | Qualification inventory, employee filter |
| `/qualifications/[id]` | Detail | Employees with this qual, expiry warnings |
| `/availability` | Dashboard | All employees' availability at a glance |
| `/availability/[employeeId]` | Manage | Add/edit availability slots + overrides |
| `/shifts` | List | Calendar/table view, filter by date/type/status |
| `/shifts/new` | Create | Form, select qualifications required |
| `/shifts/[id]` | Detail | Show assignment status, quick assign |
| `/shifts/[id]/edit` | Edit | Update shift details |
| `/sign-offs` | List | Recent incidents, filter by employee/type/date |
| `/sign-offs/new` | Create | Quick form, link to shift optional |
| `/sign-offs/[id]` | Detail | Full details, mark resolved |
| `/points` | Dashboard | Summary by employee, add form |
| `/special-services` | List | Events, filter by date/semester/type |
| `/special-services/new` | Create | Event form |
| `/special-services/[id]` | Detail | Assignments, hours, notes |
| `/special-services/[id]/edit` | Edit | Modify event details |
| `/reporting` | Hub | Links to report views |
| `/reporting/employees` | Report | Employee hours, attendance, points summary |
| `/reporting/staffing-gaps` | Report | Shifts without assignments, qualifications needed |
| `/admin/users` | Management | User creation/role assignment (ADMIN only) |
| `/admin/audit` | Viewer | Audit log search + filtering (ADMIN only) |

### 3.3 API Routes (Backend)

**Pattern:** Server actions for mutations (forms), API routes for complex queries/exports

| Route | Method | Purpose |
|-------|--------|---------|
| `POST /api/employees` | Server Action | Create employee + audit |
| `PATCH /api/employees/[id]` | Server Action | Update employee |
| `DELETE /api/employees/[id]` | Server Action | Deactivate employee |
| `GET /api/employees` | API Route | List with filters (query params) |
| `GET /api/employees/[id]` | API Route | Single employee detail |
| `POST /api/shifts` | Server Action | Create shift |
| `PATCH /api/shifts/[id]` | Server Action | Update shift |
| `POST /api/shifts/[id]/assign` | Server Action | Assign employee |
| `DELETE /api/shifts/[id]/assign/[employeeId]` | Server Action | Remove assignment |
| `POST /api/sign-offs` | Server Action | Create sign-off |
| `PATCH /api/sign-offs/[id]` | Server Action | Resolve sign-off |
| `POST /api/points` | Server Action | Add shift point |
| `GET /api/export/employees` | API Route | CSV export |
| `GET /api/export/shifts` | API Route | CSV export by date range |
| `GET /api/dashboard/metrics` | API Route | Dashboard KPIs |

### 3.4 Component Hierarchy

**Layout Structure:**
```
App
├── ProtectedLayout (session check)
│   ├── Sidebar (main nav)
│   ├── TopNav (user menu, logout)
│   └── Main Content
│       ├── Page-specific components
│       └── Shared components (DataTable, Forms, etc.)
```

**Shared Components (Reusable):**
- `DataTable`: Generic table with sorting, pagination, filtering
- `SearchBar`: Global search (employees, shifts, etc.)
- `FilterPanel`: Dynamic filter UI
- `ConfirmDialog`: Delete/action confirmations
- `Forms`: EmployeeForm, ShiftForm, etc. (all use React Hook Form + Zod)

**Page-Specific Components:**
- `EmployeeTable`, `EmployeeProfile`, `QualificationsList`
- `ShiftCalendar`, `ShiftAssignmentView`
- `SignOffList`, `SignOffForm`
- `DashboardMetrics`, `RecentActivity`

### 3.5 Design System / Styling

**Approach:**
- Tailwind CSS utility-first
- shadcn/ui components (buttons, tables, dialogs, forms, dropdowns, badges)
- Color palette: Neutral (grays), with accent color (blue/teal for CTA)
- Spacing: 4px base unit (Tailwind default)
- Typography: Geist font (Next.js default), clear hierarchy
- Icons: Lucide React (pairs with shadcn/ui)

**Example Theme:**
```css
/* Primary accent: teal/blue for operations feel */
--primary: #0891b2 (cyan-600)
--success: #10b981 (emerald-500)
--warning: #f59e0b (amber-500)
--danger: #ef4444 (red-500)
--neutral: #64748b (slate-500)

/* Typography */
h1: 2xl font-bold
h2: xl font-semibold
p: base font-normal
label: sm font-medium
```

---

## PHASE 4: STARTER IMPLEMENTATION CODE

*(Deferred until architecture is locked)*

---

## PHASE 5: FUTURE ENHANCEMENTS & AUTOMATION HOOKS

### 5.1 Rule Automation Architecture

All business rules implemented in `lib/services/rules.service.ts`:

```typescript
// Example structure (pseudocode)
export class RulesService {
  // Called after each sign-off is created
  async handleSignOffCreated(signOff: SignOff): Promise<void> {
    const recentSignOffs = await this.countRecentSignOffs(signOff.employeeId);
    if (recentSignOffs >= 3) {
      await this.triggerEscalation(signOff.employeeId);
    }
    if (this.needsEmailNotification(signOff)) {
      await this.queueEmailNotification(signOff);
    }
  }

  // Called after shift points added
  async handlePointsUpdated(employee: Employee): Promise<void> {
    const totalPoints = await this.calculateTotalPoints(employee.id);
    if (totalPoints >= 12) {
      await this.flagForWriteUp(employee.id);
    }
  }

  // Called daily (job)
  async checkQualificationExpiry(): Promise<void> {
    const expiring = await this.getExpiringQualifications();
    for (const qual of expiring) {
      await this.sendExpiryWarning(qual);
    }
  }
}
```

### 5.2 Future Features to Add (Stub Locations)

| Feature | Stub Location | Implementation Hook |
|---------|---------------|-------------------|
| 3 sign-off escalation | `rules.service.ts` | `handleSignOffCreated()` |
| Point threshold actions | `rules.service.ts` | `handlePointsUpdated()` |
| Expiration alerts | `rules.service.ts` + scheduled job | `checkQualificationExpiry()` |
| Automated email | `lib/services/email.service.ts` | Queue to Redis/database job queue |
| Conflict detection | `shift.service.ts` | `validateAssignment()` |
| Assignment suggestions | `shift.service.ts` | New endpoint `/api/shifts/[id]/suggest-assignment` |
| Real-time updates | Add Socket.io or Pusher | Broadcast on major mutations |
| Mobile app | Dedicated React Native repo | Share API layer, separate frontend |

---

## DEPENDENCIES & INSTALLATION

### 3.6 Node.js + Package Managers

**System Requirements:**
- Node.js 18+ or 20+ LTS
- npm 9+ or yarn 3+
- PostgreSQL 14+ (local or cloud, e.g., Supabase, Railway, AWS RDS)

### 3.7 Core npm Packages (Ready for Installation)

```json
{
  "dependencies": {
    "next": "^14.0.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "@prisma/client": "^5.7.0",
    "next-auth": "^5.0.0-beta.3",
    "zod": "^3.22.0",
    "react-hook-form": "^7.48.0",
    "@hookform/resolvers": "^3.3.0",
    "@tanstack/react-query": "^5.25.0",
    "tailwindcss": "^3.3.0",
    "clsx": "^2.0.0",
    "lucide-react": "^0.294.0",
    "date-fns": "^2.30.0",
    "papaparse": "^5.4.1"
  },
  "devDependencies": {
    "typescript": "^5.3.0",
    "@types/react": "^18.2.0",
    "@types/node": "^20.10.0",
    "@types/papaparse": "^5.3.0",
    "ts-node": "^10.9.0",
    "prisma": "^5.7.0",
    "@tailwindcss/forms": "^0.5.7",
    "eslint": "^8.54.0",
    "eslint-config-next": "^14.0.0"
  }
}
```

**Optional (add later if needed):**
- `@radix-ui/*` (if shadcn/ui components need customization)
- `recharts` or `chart.js` (future analytics)
- `node-cron` (scheduled jobs for exports, expiry checks)
- `nodemailer` (email sending)
- `redis` (session store, job queue)

---

## SUMMARY: READY FOR PHASE 4?

**Locked Architecture:**
✅ Next.js full-stack with Prisma + NextAuth  
✅ 9-table schema with clear relationships  
✅ 24 main routes (auth, dashboard, employees, shifts, analytics, admin)  
✅ Modular service layer for future rules  
✅ React Query for state, Server Actions for mutations  
✅ shadcn/ui + Tailwind for clean UI  
✅ MVP scope: all core CRUD + dashboard + reporting  

**Next Step:** Ready to generate:
1. Prisma schema file
2. Database seed data
3. Starter pages + layouts
4. Core services
5. API routes + Server Actions
6. Component skeletons
7. Auth setup (NextAuth config)

**Ready?** Reply with:
- Any schema adjustments
- Any route/page changes needed
- Confirmation to proceed with full codebase generation
- Database choice (Supabase / local PostgreSQL / other)
