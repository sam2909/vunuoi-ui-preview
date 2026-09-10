# VUNUOI Implementation Guide

> Status: living implementation contract for the current UI prototype.
>
> Purpose: give Codex enough product, UX, data, permission, and backend-rule context to implement the real VUNUOI app without guessing from HTML alone.

## 1. Source of truth

Use these three sources together:

1. `index.html` = current product/UI prototype.
2. `IMPLEMENTATION.md` = intent, rules, data contracts, and constraints behind the prototype.
3. Real VUNUOI application source = actual architecture, models, APIs, auth, DB schema, and deployment constraints.

Before implementing anything, inspect the real app first. Do not blindly port prototype HTML/CSS into production. Adapt the agreed UX to the existing architecture.

## 2. Product principles already decided

### DECIDED

- Mobile-first means responsive from mobile upward, not mobile-only.
- Homepage is a quick operating view, not a dense dashboard.
- Cross-pond comparison is important.
- The homepage should answer, in order:
  1. Which farm/season am I looking at?
  2. How are the current ponds doing?
  3. What do I need to do today?
- Do not add extra top-level navigation unless clearly necessary.
- Do not add a separate top-level `Ao nuôi` page just because ponds exist. Pond navigation should be reachable from homepage and `Mùa vụ -> Khu -> Ao`.
- Avoid generic status labels that cannot be defined deterministically.
- Never invent live sensor intelligence when the system does not have live sensors.

## 3. Core hierarchy

### DECIDED

Logical hierarchy:

`Farm -> Season -> Zone (optional) -> Pond`

- `Zone` / `Khu nuôi` is optional.
- If a season has no zones, render ponds directly under the season.
- Do not force every season to have zones.
- The current UI example is:
  - PewPewFarm
    - Mùa 27
      - Khu A
        - N003
        - N201
      - Khu B
        - N302

### IMPORTANT DOMAIN NOTE

Do not assume shrimp age from season start date.

Shrimp age must be calculated from each pond's stocking date / `pond_stocked_at` equivalent. Ponds in the same season may have different ages.

## 4. Homepage

### DECIDED

Homepage is considered a baseline and should stay compact.

Current information structure:

1. App chrome/header
2. Farm context
3. Active season block
4. Optional zones
5. Pond summary cards
6. `Sổ hôm nay`

### Farm context

Show useful context, not redundant labels like:

- `Tổng quan vận hành`
- repeated `Hôm nay`
- `Dashboard`

Preferred information is actual scope and status, e.g.:

`PewPewFarm`
`Mùa 27 · 3 ao · 2 khu`

A lightweight summary such as `3 việc cần làm` may link/scroll to the task section.

Do not build a separate alert system that duplicates `Sổ hôm nay`.

## 5. Pond card

### DECIDED

Homepage pond cards should stay compact and comparable.

Current summary fields:

- Pond name/code
- Latest measured size
- Age at the time of the latest size measurement
- Current pond age
- Growth rate
- Data freshness / actionable status

Example size block:

- `34,8 con/kg`
- `ngày 90` = age when that size was measured

Separate current age:

- `92 ngày`

This separation is important because otherwise users may mistake the measured size as being from the current age.

### STATUS RULE

Do not use a vague `Ổn` badge unless there is a documented deterministic rule for it.

Prefer explicit states such as:

- `Đã cập nhật`
- `Đến lịch đo`
- `Thiếu cập nhật`

Even better, where possible show exact freshness:

- `Size · 2 ngày trước`
- `Môi trường · sáng nay`

Backend should decide these states from known data/rules. Frontend should render them.

## 6. Pond detail navigation

### DECIDED DIRECTION

Pond cards should be clickable.

Example:

`N003 card -> Pond detail N003`

Pond Detail is an operating view for one pond. Its hierarchy should answer:

1. What is the pond's current known state?
2. Is there something that needs action now?
3. How has growth changed between real measurements?
4. What feed has been recorded around those measurement periods?
5. What environmental values were last recorded?
6. What recently happened in the pond?

Do not style ordinary factual information like CTAs. Current actionable work should be visually isolated from passive facts.

### Growth measurement deltas

### DECIDED

For each real size/weight measurement after the first available measurement, derive and display comparison with the immediately previous real measurement:

- elapsed days
- change in `g/con`
- change in `con/kg`
- average daily gain over that interval

Example:

`03/09 -> 06/09`
`26.11 -> 29.33 g/con`
`38.3 -> 34.1 con/kg`
`+3.22 g/con · giảm 4.2 con/kg · 3 ngày · +1.08 g/con/ngày`

Do not compare against an arbitrary older measurement when a directly adjacent measurement exists.

Do not interpret `con/kg` decreasing as a negative outcome. It generally corresponds to animals becoming larger, so wording/icons must avoid implying that the decrease itself is bad.

### Feed-growth measurement intervals

### DECIDED DIRECTION

Feed-growth context should be organized around real size-measurement intervals rather than pretending every day has a complete efficiency result.

After a size measurement, start an open interval conceptually:

`latest size measurement -> next real size measurement`

While the interval is still open, it may show deterministic facts such as:

- interval start date
- elapsed days
- total feed recorded since that measurement
- status such as `Đang chờ size mới`

Do not manufacture a weight gain or efficiency result before the next real size measurement exists.

When the next size measurement is saved, the interval can be closed and the UI may compare recorded feed with measured growth for that same interval.

Do not calculate or label a value as FCR unless the production data model contains the required trustworthy biomass/survival/population inputs and the backend rule is explicitly defined.

## 7. Sổ hôm nay

### DECIDED

`Sổ hôm nay` is a daily operating notebook/task list.

It is not another hierarchy level beside Season/Zone/Pond. Task cards describe work to perform within a scope.

Task card hierarchy must be:

1. **Action** = primary text
2. **Scope** = metadata
3. **Source/rule** = metadata
4. Due time/date = metadata

Good:

`Đo size`
`Ao N302 · Theo lịch · Lần cuối 06/09`

Bad:

`N302 · Đo size`

Bad because it visually flattens Pond and Zone scopes into peer titles such as `Khu A · ...` and `N302 · ...`.

### Task sources

Support these conceptual sources:

1. `recurring_template`
   - user-configured recurring work
   - e.g. measure environment daily at 07:00

2. `manual_task`
   - user-created note/work item
   - e.g. `Mai kiểm tra lại màu nước Khu B`

3. `derived_schedule`
   - deterministic tasks generated from known data/rules
   - e.g. pond size measurement is overdue because cycle = 3 days and last measurement was 4 days ago

### IMPORTANT

Derived tasks must only come from deterministic data/rules.

Allowed:

- `N302 đến lịch đo size`

Not allowed without suitable real data/sensors/rules:

- `NH3 đang cao`
- `Nước đang xấu`
- any inferred diagnosis presented as fact

## 8. Carry-forward behavior

### DECIDED

Unfinished manual/eligible tasks may carry forward to the next day when this setting is enabled.

UI should label carried work clearly, e.g.:

`Từ hôm qua`

Tasks must not silently disappear at midnight.

## 9. Task actions

### DECIDED

Tasks should be actionable directly from homepage.

Examples:

- `Đo môi trường buổi sáng` -> open Environment input with Khu A / relevant ponds preselected
- `Đo size` -> open Size input with N302 preselected
- manual task -> open task detail/editor

Backend/API task payload should ideally provide something equivalent to:

```json
{
  "id": 123,
  "title": "Đo size",
  "source": "derived_schedule",
  "scope_type": "pond",
  "scope_id": 302,
  "action_type": "open_size_input",
  "target_scope_type": "pond",
  "target_scope_id": 302
}
```

Frontend must not parse task title text to guess the action.

## 10. Completing tasks

### DECIDED

Data-entry tasks should complete automatically when the underlying required data is successfully entered.

Example:

`Đo size N302 -> submit valid size entry -> matching task completes`

Do not force the user to:

1. enter the data
2. navigate back
3. manually tick the same task

Manual tasks may still use explicit checkbox/completion action.

## 11. Input screen

### DECIDED DIRECTION

Input must support fast daily work and batch entry.

Important targets:

- feed
- environment
- size

Feed should support multi-pond and multi-day entry where practical.

### Feed backfill rules

### DECIDED

- A multi-day feed backfill form lists only dates that do not yet have a completed feed-state record.
- A date with feed rows is complete and must not be listed as missing.
- A date explicitly recorded as intentional no-feed is also complete and must not be listed as missing.
- Blank dates remain missing.
- Default to the 7 most recent missing dates; allow the user to request a smaller subset where useful.
- Copying the previous displayed date must copy its complete state:
  - feed rows, or
  - intentional no-feed plus its reason.
- Normal feed input is `kg/feeding × feeding count = daily total`. Daily total is derived, not a competing third source of truth.
- Intentional no-feed is a day-level operating state, not a feed type.
- Feed types have a managed lifecycle: add, edit, hide/reactivate, and delete only when never referenced. Referenced types should be hidden rather than hard-deleted.

Task deep-links should open the relevant input type and preselect the correct scope.

Do not make users repeatedly drill through Farm -> Season -> Zone -> Pond for routine input when task context already provides the scope.

## 12. Admin and farm switching

### DECIDED

Admin is a capability/control, not a farm-view status.

Do not display text like:

`Trang trại đang xem · Admin`

This mixes two unrelated concepts.

Use separate controls:

- Farm selector
- Admin button
- Settings button

### Farm selector

Render only when the user has permission to switch between multiple farms.

Options must come only from authorized farms.

Do not expose all farms blindly.

Conceptual permission field:

`user.can_switch_farm = true`

### Admin button

Render separately only for users with appropriate admin capability.

Conceptual field:

`user.is_admin = true`

The real app may already use roles/permissions differently. Reuse the real authorization system rather than adding these exact fields if equivalent capability checks already exist.

## 13. Settings

### DECIDED

Settings owns operating rules and notification preferences, not the Farm/Season/Zone/Pond structure itself.

Current conceptual settings:

### Sổ hôm nay

- recurring environment measurement time
- size measurement cycle
- carry unfinished tasks to next day

### Notifications

- morning summary time
- reminder lead time
- enabled notification types

### Admin-only

- current farm / farm switching where appropriate
- user management
- role/permission management

Farm/Season/Zone/Pond creation and management belongs to season/farm management, not generic Settings.

## 14. Backend responsibility vs frontend responsibility

### BACKEND SHOULD OWN

- authorization
- farm access scope
- active season lookup
- pond age calculation from stocking date
- latest measurement selection
- growth calculations/business rules
- adjacent-measurement delta calculations
- feed aggregation for measurement intervals
- task generation/synchronization
- recurring schedules
- derived overdue rules
- data-freshness status
- task completion after data entry
- notification scheduling

### FRONTEND SHOULD OWN

- rendering hierarchy
- responsive layout
- navigation
- task action routing from explicit action metadata
- forms and validation UX
- visual state

### DO NOT

Do not duplicate core business rules in frontend if backend is the source of truth.

## 15. Suggested homepage payload

This is conceptual, not a mandatory exact schema:

```json
{
  "selected_farm": {
    "id": 1,
    "name": "PewPewFarm"
  },
  "can_switch_farm": true,
  "is_admin": true,
  "active_season": {
    "id": 27,
    "name": "Mùa 27",
    "started_at": "2026-06-01",
    "zones": [
      {
        "id": 1,
        "name": "Khu A",
        "ponds": []
      }
    ]
  },
  "today_tasks": [],
  "today_summary": {
    "pending_tasks": 3
  }
}
```

If there are no zones, `active_season.ponds` or equivalent may be rendered directly under the season.

Adapt this to the real API/domain model.

## 16. Navigation baseline

### DECIDED

Current main navigation direction:

- Hôm nay
- Nhập
- Báo cáo
- Mùa vụ
- Công cụ

Settings can be accessed through the gear button.

Admin can be a separate button/capability area.

Do not add another top-level nav item without checking whether an existing route/hierarchy already solves the use case.

## 17. Reports

### DECIDED DIRECTION

Reports should support:

- by pond
- by season
- pond comparison
- selectable time range

Likely report domains:

- growth
- feed
- FCR
- environment

Homepage should not absorb these detailed charts.

## 18. Notification philosophy

### DECIDED

Notifications should help answer `Hôm nay làm gì?` rather than spam raw data.

Examples:

- morning summary of today's work
- due task reminder

Avoid notifications pretending to know conditions that are not actually measured.

## 19. Prototype data is not production truth

### IMPORTANT

Names and values in `index.html` and `pond-detail.html` are demo data only, including:

- PewPewFarm
- Mùa 27
- Khu A / Khu B
- N003 / N201 / N302
- size values
- age values
- task times
- feed totals used to demonstrate the interval UI

Do not hardcode these into production.

## 20. Implementation workflow for Codex

When asked to implement a prototype feature in the real VUNUOI source:

1. Read this file.
2. Read the relevant prototype HTML.
3. Inspect existing production routes, controllers/services, models, DB schema, auth/permissions, API resources, frontend components, and tests.
4. Map prototype concepts to existing domain names rather than creating duplicates.
5. Identify migrations only if the existing schema genuinely lacks the required concept.
6. Keep backend business logic centralized.
7. Implement responsive UI matching the intended hierarchy, not necessarily exact prototype CSS.
8. Add/update tests for deterministic rules.
9. Explain any mismatch between prototype assumptions and current source before making a disruptive architecture change.

## 21. Status legend

Use these labels when extending this file:

- `DECIDED` = product/UX behavior agreed and should be preserved unless explicitly changed.
- `TODO NEXT` = intended next implementation/design target.
- `DIRECTION` = agreed direction but details may still change.
- `DO NOT` = known anti-pattern or behavior we explicitly rejected.
- `PROTOTYPE ONLY` = demo representation, not final data/schema/API.

## 22. Current next target

### TODO NEXT

Continue refining the **Pond Detail** operating flow, then move into the fast input flow with pond/task scope already preselected.

Current Pond Detail questions:

1. Pond is at what age and current measured size?
2. What changed between adjacent real measurements?
3. Is there an action due now?
4. How much feed has been recorded recently and since the latest size measurement?
5. Is the current feed-growth interval still open or can it be evaluated using a new real measurement?
6. What environmental measurements were recently recorded?
7. What tasks/notes/history are associated with this pond?
8. What deeper analysis belongs in Reports rather than this operating view?

Do not turn Pond Detail into a wall of cards. Prioritize current state, action, adjacent-measurement change, and recent factual context.

---

Last updated: 2026-09-11
