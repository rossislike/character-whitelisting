# Deepesh Character Whitelist Part 4 - On-Screen Text

Source: `2026-07-23-Deepesh-character-whitelist-p4.mp4`

Scope: Visible product and design content through 00:21:16. Text is lightly normalized for capitalization, spacing, and obvious screen-share artifacts.

## Proposed upload and cleanup flow

Text entered in the Teams chat:

> User uploads the files - Goes to blob
>
> We run sanitization endpoint in collateral service
>
> This endpoint copy this file and creates a backup at a different location
>
> We take the original file - run regex and sanitize it
>
> assuming collateral has access to blob location
>
> and return success
>
> Phase 1
>
> in phase 1.1 - check user access to file
>
> phase 2 - Add logging

## Funding data dictionary

The shared Excel workbook showed a `Dictionary` tab with these principal columns:

- Field
- Description
- Format
- Example
- Valid Values
- Investor or program applicability columns

The discussion described the workbook as the source for the funding file's possible columns and data types. The participants estimated about 196 columns.

## Azure DevOps story 23509

Title: `Resi: Special Characters in Funding File`

Visible description:

> Some of the special characters in funding files are causing issues when uploading to Promerit.
>
> Please see below of a breakdown of the special characters that are allowed in ProMerit.

### Default whitelist

```text
A-Z a-z 0-9
Ampersand &
Apostrophe '
At @
Backtick `
Caret ^
Comma ,
Colon :
Curly braces {}
Dash -
Dollar $
Exclamation !
Hash #
Parentheses ()
Period .
Pipe |
Plus +
Quotes "" (is a special Word quote)
Semicolon ;
Slashes \ /
Space
Square brackets []
Star *
Tilde ~
Underscore _
Wildcard %
```

### Description whitelist

The description whitelist repeats the field/default characters and adds:

```text
Angle brackets < >
Question mark ?
Carriage returns
Tabs
```

The visible note for angle brackets says their use in account names is still under review and suggests not using them in account key fields until confirmed.

## Rev 10 plan summary shown during the call

Visible before and after comparison:

| Before | After |
| --- | --- |
| Single file-wide allow-list per column moved to v2 | `FIELD` and `DESCRIPTION` per column in v1 |
| No schema mapping | Column type comes from the Collateral schema; unknown columns default to `FIELD` |
| No description delta | `DESCRIPTION = FIELD + < > ? \r \t` based on ProMerit |

Visible open items attributed to Srini:

1. Add the payoff strip/sanitize behavior as a configured rule after receiving the code pointer.
2. Obtain Srini's final review after the description profiles and payoff rule are present.

Other visible plan text:

- Seed rules version bumped to `2026.07.3`.
- Handoff, CSV/Excel fidelity, PII, and per-column `FIELD`/`DESCRIPTION` were marked complete.
- Collateral sanitize service plus Funding/Digital Connect access adapters was the next implementation item.
- Configure the UI for both `FIELD` and `DESCRIPTION` lists.
- Cross-service access must not trust a browser-supplied blob path.

## Short-lived access design

Visible simplified design:

> Collateral does not have permanent access to Funding or Digital Connect storage.
>
> When the UI asks Collateral to clean a file, Collateral asks the owner, Funding or Digital Connect: "Give me temporary access to document X for this user."
>
> If that checks out, the owner hands back two short-lived links, SAS URLs:
>
> - one to read the original file
> - one to write the cleaned file
>
> Collateral uses those links to pull the file from Azure Blob, clean it, and put the cleaned copy back. When the links expire, Collateral can no longer touch those blobs.
>
> The UI only sends a document ID - never the file location or the links.

## Implementation plan text visible near 20:10

- The UI uploads the file and obtains a document ID.
- The UI invokes `POST /file-cleanup/sanitize`.
- Collateral requests cleanup authorization from the file owner.
- The owner authorizes the document and issues short-lived read/write access.
- Collateral parses the file, applies per-column allow-list rules, writes the sanitized file, and records remediation information.
- The UI keeps submit/process blocked until cleanup succeeds, then continues normal validation.

