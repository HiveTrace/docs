---
title: Data cleaning
description: Configure detection, pseudonymization, and masking of sensitive data in messages and files.
sidebar:
  order: 6.5
---

The **Data cleaning** section configures sensitive-data processing for each application. Saved settings apply to new messages and files.

Open **Applications**, select the application ID, and choose **Data cleaning**.

![Application data-cleaning settings](/docs/images/hivetrace-monitoring/dataclean/dataclean-settings-overview-4-3.png)

## Built-in patterns

The 16 built-in patterns are Address, Bank card, Card code, Domain name, Email, Tax ID, IP address, Tax registration reason code, Names, Primary State Registration Number, Individual Entrepreneur Registration Number, Passport, Phone, Individual Insurance Account Number, Access token, and Link.

Each card has two switches: **Input** checks user requests and **Output** checks model responses. The information icon opens the pattern description.

![Built-in patterns and direction switches](/docs/images/hivetrace-monitoring/dataclean/dataclean-basic-patterns-4-3.png)

## Custom and allowed patterns

**Custom patterns** add team-specific regular expressions. Use **Add pattern**, then enable Input and/or Output. The three-dot menu provides edit and delete actions.

**Allowed patterns** define exceptions for values that resemble sensitive data but are valid in your workflow.

![Custom and allowed patterns](/docs/images/hivetrace-monitoring/dataclean/dataclean-custom-allowed-patterns-4-3.png)

![Pattern cards and add controls](/docs/images/hivetrace-monitoring/dataclean/dataclean-pattern-groups-4-3.png)

## Processing mode

Input and output can use different modes:

- **Masking** replaces the detected value with `X` characters.
- **Pseudonymization** replaces it with a stable label such as `NAME_1` or `EMAIL_1`.

## File processing

**Incoming files** and **Outgoing files** control text extraction and inspection for user attachments and model-produced files. **Detection only** means entities are reported without modifying the file.

![Processing mode and file processing](/docs/images/hivetrace-monitoring/dataclean/dataclean-file-processing-4-3.png)

## Inspection depth and source storage

Message and file inspection depth controls how much context is analyzed. Deeper inspection can improve contextual detection but costs more time and resources.

Source-storage switches retain both original and cleaned messages for comparison in analytics. Enable them only when permitted by your access and retention policies.

![Inspection depth and source-message storage](/docs/images/hivetrace-monitoring/dataclean/dataclean-depth-storage-4-3.png)

## Test data cleaning

1. Enter a sample in **Source text**.
2. Click **Clean**.
3. Compare **Cleaned text (Input)** and **Cleaned text (Output)**.
4. Verify pseudonymized labels and masked values.
5. Review **Detected sensitive data** using the Incoming and Outgoing tabs.

![Text-cleaning test](/docs/images/hivetrace-monitoring/dataclean/dataclean-text-test-4-3.png)

The result table shows the entity type, source text, and cleaned text. Use **Column settings** to select visible columns.

![Detected sensitive-data table](/docs/images/hivetrace-monitoring/dataclean/dataclean-detected-data-4-3.png)

## Recommended setup order

1. Enable the required built-in patterns and directions.
2. Add custom patterns and exceptions.
3. Select message-processing modes.
4. Configure files, inspection depth, and source storage.
5. Test the configuration.
6. Send a test request and verify it in **Session analytics**.
