---
title: "Session analytics"
sidebar:
  order: 16
---

**Session analytics** is a workspace for investigating requests, model responses, violations, files, and tool calls. Header counters show all records and records with violations; `live` indicates streaming updates.

![Session analytics overview](../../../../../assets/img/en/sessions-overview-4-4.webp)

## Search and export

Global search covers the full record ID, session ID, user ID or name, and message text. Enter 3–256 characters; use the advanced filter for an exact UUID. **Download sessions** exports the current filtered result. Verify period, application, and filters because the file can contain original messages and personal data.

## Quick filters

- **Today** — records for the current day;
- **Violations** — records with at least one violation;
- **With attachments** — records containing files;
- **Data cleansing · 7 days** — DataClean violations from the latest seven days.

## Period

![Sessions - period filter](../../../../../assets/img/en/sessions-period-filter-4-4.webp)

Choose **Today**, **7 days**, **30 days**, **All time**, or a custom calendar range. Month arrows navigate the calendar. Select both boundaries and then **Apply**. Future dates are disabled.

## Application

![Sessions - application filter](../../../../../assets/img/en/sessions-application-filter-4-4.webp)

The list shows applications, including deleted ones, as well as the number of matching records. Search narrows the list. **All** removes the application restriction; selecting an application updates both chart and table.

## User groups

![Sessions - user groups filter](../../../../../assets/img/en/sessions-user-groups-filter-4-4.webp)

The list shows user groups. One can select multiple groups — the selected groups are added to the field. Search narrows the list. A group can be removed from the list by clicking the **×** next to its name. The filter becomes active once an application is selected; until then, the field is disabled.

## Violation

![Sessions - violation filter](../../../../../assets/img/en/sessions-violation-filter-4-4.webp)

Options are **All**, **Any violation**, **No violations**, **Safety policy** (Guardrail), **Custom policy**, **Data cleansing**, and **Language mismatch**. The number on the right is the matching count for the current selection.

## Saved filters

![Sessions - saved filter sets](../../../../../assets/img/en/sessions-saved-filters-4-4.webp)

This section stores named condition sets; the button badge is their count. Apply at least one filter before saving. Selecting a card reapplies its conditions. Available set actions let you rename, update from the current conditions, or delete it.

## Advanced filter

![Sessions - advanced session filter](../../../../../assets/img/en/sessions-advanced-filter-4-4.webp)

- **Session ID**, **Record ID**, and **User ID** perform exact identifier filtering;
- **User name** filters by display name;
- **Direction** selects all, `in`, or `out`;
- **Message contains** matches a text fragment;
- **With attachments** requires files;
- **With tool calls** requires tool calls;
- **Block type** selects `Blocked`, `Would block`;
- **Block stage** selects `Before model call`, `After model call`;
- **Block reason** selects `Guardrail Policy`, `Custom Policy`, `Data Cleansing`.

The button badge shows active advanced conditions. **Apply** updates results; **Reset** clears this section.

## Violation trend

The chart follows current filters. **Total for selected period** is the sum, while **Peak** shows the largest daily value and date. Selecting a bar adds that day as a slice. **Collapse** hides the chart.

## Records table

Records load in batches; the label above the table shows loaded and total counts. Selecting a cell value filters by that value.

![Sessions table sorting](../../../../../assets/img/en/sessions-records-sorting-4-4.webp)

Sessions can be sorted by date — ascending or descending. By default, descending sorting is applied. If `Newest first` is selected, descending sorting is applied; if `Oldest first` is selected, ascending sorting is applied.

![Record identifiers and column settings](../../../../../assets/img/en/sessions-columns-settings-4-4.webp)

**Customize Columns** shows the visible-column count and toggles record ID, session, user, application, message, agents/tool calls, files, violation, direction, and date.

![Sessions table: identifiers, user, and application](../../../../../assets/img/en/sessions-record-identifiers-4-4.webp)

Record **ID** links to details. **Session** is the client session identifier. **User** contains supplied name and identifier. **Application** identifies the event source.

![Table: content, tool calls, violations, and date](../../../../../assets/img/en/sessions-record-content-4-4.webp)

**Message** is a shortened request or response. **Agents · Tools** shows related tool-call count. **Files** displays the number of files attached to the session. **Violation** identifies the triggered check. **Direction** is `in` or `out`. **Date** uses local date and time. Scroll horizontally to reveal columns that do not fit.

Select a record ID to open [Session details](./session-details.md).

## Access to the page

To view and filter sessions, the following permissions are required:

- Area **Sessions and messages** — access level of at least `Read`.

To filter sessions by application, the following permissions are required:

- Area **Sessions and messages** — access level of at least `Read`;
- Area **Applications** — access level of at least `Read`.

To filter sessions by user groups, the following permissions are required:

- Area **Sessions and messages** — access level of at least `Read`;
- Area **Applications** — access level of at least `Read`;
- Area **End users** — access level of at least `Read`.

To export sessions, the following permissions are required:

- Area **Sessions and messages** — access level of at least `Read`;
- Enable the **Export sessions** granular permission.

If the user does not have access to view the page, `403 — Access denied` is displayed.
