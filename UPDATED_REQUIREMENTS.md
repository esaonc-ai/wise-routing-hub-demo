# WISE Routing Hub — Updated Production Requirements

## Mandatory operating model
- Facility and retailer masters are centrally administered, scalable, and never require frontend changes to add entities.
- Users have many-to-many facility, retailer, and account assignments; dashboard data is always scoped to those assignments.
- Supervisors/admins can switch between **My Assigned Orders** and **All Accessible Orders**.
- Active master records are selectable; inactive records remain auditable but cannot be newly assigned.

## Updated information architecture
- **Auth:** login, MFA, access request, reset password.
- **Dashboard:** personalized KPIs, assignment-scope toggle, new-order badges, attention queue.
- **Orders:** saved multi-select filters, bulk routing, history and audit detail.
- **Facilities / Retailers:** master lists, detail pages, lifecycle management.
- **Settings:** profile, multi-facility / retailer / account assignments, notifications, shift, display preferences.
- **Admin:** users, roles, master data, assignment bulk updates, WISE sync health and logs.

## Updated key user flows
### Access request
Sign up → select multiple facilities, retailers/accounts → pending approval → admin approves assignments → invite/password set → MFA/login.

### Assignment-aware routing
Login → load permitted facilities/retailers/accounts → select current facility or scope → saved per-facility filters restore → route only accessible orders.

### Master data maintenance
Admin → Facility or Retailer Master → add/edit/category → activate/deactivate → record immediately available/removed from assignment pickers; historical orders retain master reference.

### New order detection
WISE sync normalizes order → assignment engine determines eligible users → realtime event reaches matching clients → new-order badge/toast → urgency engine applies business-calendar and carrier-cutoff deadline rules.

## Production data model
```sql
CREATE TABLE facilities (
  id UUID PRIMARY KEY, code TEXT UNIQUE NOT NULL, name TEXT NOT NULL,
  country_code TEXT, timezone TEXT NOT NULL, active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(), updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE TABLE retailers (
  id UUID PRIMARY KEY, code TEXT UNIQUE, name TEXT NOT NULL,
  customer_category TEXT, business_unit TEXT, shipping_method TEXT,
  active BOOLEAN NOT NULL DEFAULT true, created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE TABLE user_facilities (
  user_id UUID REFERENCES users(id), facility_id UUID REFERENCES facilities(id),
  access_status TEXT NOT NULL DEFAULT 'approved', assigned_by UUID REFERENCES users(id),
  assigned_at TIMESTAMPTZ NOT NULL DEFAULT now(), PRIMARY KEY(user_id, facility_id)
);
CREATE TABLE user_retailers (
  user_id UUID REFERENCES users(id), retailer_id UUID REFERENCES retailers(id),
  access_status TEXT NOT NULL DEFAULT 'approved', assigned_by UUID REFERENCES users(id),
  assigned_at TIMESTAMPTZ NOT NULL DEFAULT now(), PRIMARY KEY(user_id, retailer_id)
);
CREATE TABLE user_accounts (
  user_id UUID REFERENCES users(id), account_id UUID REFERENCES accounts(id),
  access_status TEXT NOT NULL DEFAULT 'approved', PRIMARY KEY(user_id, account_id)
);
CREATE TABLE user_preferences (
  user_id UUID PRIMARY KEY REFERENCES users(id), theme TEXT NOT NULL DEFAULT 'light',
  active_scope TEXT NOT NULL DEFAULT 'assigned', saved_filters JSONB NOT NULL DEFAULT '{}',
  working_shift JSONB, notification_preferences JSONB NOT NULL DEFAULT '{}', updated_at TIMESTAMPTZ DEFAULT now()
);
CREATE TABLE integration_runs (
  id UUID PRIMARY KEY, source TEXT NOT NULL, status TEXT NOT NULL, started_at TIMESTAMPTZ,
  completed_at TIMESTAMPTZ, last_success_at TIMESTAMPTZ, records_received INT DEFAULT 0, error_detail TEXT
);
```

## API additions
- `GET/POST/PATCH /api/admin/facilities`
- `GET/POST/PATCH /api/admin/retailers`
- `GET/PATCH /api/users/me/assignments`
- `GET/PATCH /api/users/me/preferences`
- `GET /api/orders/new?since=1h|4h|24h|48h`
- `GET /api/admin/integration-runs`

## Operational controls
Deadline scoring uses facility business calendars, holidays, retailer SLA rules, and carrier cutoff windows. Audit records are append-only. Sync failure or stale data is critical severity and is surfaced in Admin and in notifications.
