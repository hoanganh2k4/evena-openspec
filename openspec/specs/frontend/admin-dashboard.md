# Evena — Admin Dashboard Components

**Version:** 1.0 · **Date:** 2026-05-16

---

## Activity Log Page

**File:** `app/dashboard/admin/activity-log/page.tsx`

### Sparkline (hourly mini line chart)

Each stat card in the Activity Log header has a sparkline showing hourly event counts.

```tsx
<LineChart data={points}>
  <Line
    type="monotone"
    dataKey="v"
    stroke={color}
    strokeWidth={1.5}
    dot={false}
    activeDot={{ r: 3, fill: color, stroke: '#fff', strokeWidth: 1.5 }}
  />
  <ReTooltip
    formatter={(v: unknown) => [`${v ?? 0}`, 'events'] as [string, string]}
    labelFormatter={() => ''}
    cursor={{ stroke: color, strokeWidth: 1, strokeDasharray: '3 3' }}
  />
</LineChart>
```

**Type rule:** `formatter` must return `[string, string]` cast explicitly — returning `[{}, string]`
or `[number, string]` will fail `tsc --noEmit` (Recharts `Formatter<ValueType, NameType>` requires
`ReactNode` at position 0, and `{}` is not assignable to `ReactNode`).

### Bar chart — hour filter interaction

`onHourClick(hour)` must call `setActiveSavedFilter(-1)` to deactivate any currently highlighted
saved-filter chip. Without this, the saved filter chip stays visually selected even though the
hour filter has overridden it.

### Investigate button

`onInvestigate()` derives its filter window from actual spike hours (`isSpike === true` in
`hourlyData`), NOT from a hardcoded action type.

**Algorithm:**
1. Collect all hours where `isSpike === true`
2. Find index of first and last spike
3. Compute `from` = `baseDate + firstSpikeIndex hours`, `to` = `baseDate + (lastSpikeIndex + 1) hours`
4. Set `fromFilter`, `toFilter`, clear `actionFilter`, reset `activeSavedFilter` and `page`
5. Fallback (no spikes): show today's full log with no action filter

**Why:** The spike detection and the action type filter are orthogonal. The Investigate button
should surface the time window of the anomaly, letting the user browse all action types within it.

### Saved filter chips

When any of the following occur, `setActiveSavedFilter(-1)` MUST be called to deactivate the
currently selected saved filter:
- User clicks a bar in the hourly bar chart
- User clicks Investigate
- User changes the action type dropdown
- User changes the entity type dropdown

---

## AnomalySignalsPanel

**File:** `src/components/AdminDashboard/AnomalySignalsPanel.tsx`

Surfaces anomaly signals derived from SSE activity data. Displayed on the admin main dashboard.

---

## SSEHealthPanel

**File:** `src/components/AdminDashboard/SSEHealthPanel.tsx`

Monitors SSE connection health and subscriber counts. Displayed on the admin main dashboard.

Both panels are exported from `src/components/AdminDashboard/index.ts`.
