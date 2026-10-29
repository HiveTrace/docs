---
title: "Session details"
sidebar:
  order: 17
---

Open session details by clicking a record ID in the **Session analytics** table. The page preserves how HiveTrace processed the message and helps you determine which checks fired and why the request was allowed or blocked.

![Blocked session details overview](../../../../../assets/img/en/session-details-overview-4-4.webp)

The page header contains:

- the record ID in the title and breadcrumbs;
- a **Violation detected** banner listing the modules that fired;
- a **Request blocked** banner when processing was stopped;
- the blocking reasons and enforcement layer, such as `gateway` or `claude_code_hook`.

The page contains the `Back to session list` link that leads to the **Session Analytics** section.

A detected violation does not always mean that a request was blocked. The decision depends on the integration type, operating mode, and application settings active at processing time.

## Overview

The **Overview** tab contains the record's main attributes:

- session ID;
- user, when one could be identified;
- application, date, and time;
- integration method and operating mode;
- message direction: `in` for a user request and `out` for a model response;
- original message.

An unavailable or unspecified value is shown as a dash. Use the tabs below the status banners to move between detail sections.

## Tool calls

The **Tool calls** tab covers tool-based workflows, including Claude Code Hooks. It shows each tool name, processing status, and call time.

![Tool calls with a blocked call expanded](../../../../../assets/img/en/session-details-tool-calls-4-4.webp)

Statuses distinguish completed operations from blocked ones. Click a call card or its right-hand chevron to expand it. A blocked operation displays its reason and input arguments; in the example, DataClean stopped an `Edit` call. Other cards can be expanded and collapsed in the same way.

## Files

The **Files** tab lists attachments associated with the record and their validation results.

![File card with detected DataClean entities](../../../../../assets/img/en/session-details-files-4-4.webp)

Each card shows the file name, size, date, extension, and detected entity types with match counts. Actions appear on the right:

- the eye icon opens a preview;
- the download icon saves the original file;
- **×** closes the preview dialog.

![Text file preview](../../../../../assets/img/en/session-details-file-preview-4-4.webp)

Preview lets you inspect the context of detected entities without downloading the file. Because content can contain personal or other sensitive data, open it only when you have the required access. Use download when the browser cannot preview the format.

## Data cleansing

The **Data cleansing** tab explains how personal data in the message was processed.

![Data cleansing result and detected-pattern table](../../../../../assets/img/en/session-details-data-cleansing-4-4.webp)

The tab shows:

- the effective inspection depth;
- whether personal data was detected;
- the text after masking or another configured transformation;
- the number of detected patterns;
- a table with each type, name, and match count.

Compare this tab with **Validation configuration**: Data cleansing shows the effective result, while Validation configuration shows the settings applied to the record.

## Censors

The **Censors** tab combines Safety Policy and Custom Policy results.

![Safety Policy and custom censor results](../../../../../assets/img/en/session-details-censors-4-4.webp)

The **Safety Policies** card indicates whether built-in threats such as prompt injection or jailbreak were detected. **Custom Policies** reports message and file results separately and identifies the validation method, such as model classifier or blacklist. **Clean** and **Detected** badges reveal which censor caused the violation.

## Validation configuration

The **Validation configuration** tab is a snapshot of the settings used to process this record. Editing current application settings does not rewrite information for an earlier session.

![Validation configuration applied to the record](../../../../../assets/img/en/session-details-validation-config-4-4.webp)

The section starts with the configuration source, such as **Web interface**, and then lists module-specific values:

- whether a module was enabled globally and in application settings;
- Custom Policy prompts and patterns;
- whether DataClean pseudonymization was enabled;
- processing type and configured inspection depth;
- DataClean patterns.

Use the copy icon next to a pattern to place its value on the clipboard. During an investigation, compare this snapshot with the effective results in **Data cleansing** and **Censors**.

## Access to the page

To view the session details page, the following permissions are required:

- Area **Sessions and messages** — access level of at least `Read`.

To view message content inside sessions, the following permissions are required::

- Area **Sessions and messages** — access level of at least `Read`;
- Enable the **Read session content** granular permission.

To preview and download attached files, the following permissions are required:

- Area **Sessions and messages** — access level of at least `Read`;
- Enable the **Download analysis files** granular permission.

If the user does not have access to view the page, `403 — Access denied` is displayed.
