
# ProMerit Whitelisting (File Clean up service)

1. The user uploads a file. This part does not change at all.
2. The user clicks submit (funding) or process (payoff).
3. The owning service Funding or Digital Connect first checks the user is allowed to change that request. This check already exists.
4. That service then calls the Collateral service and asks it to clean the file.
5. Collateral makes a copy of the original file and puts it in an archive folder. That copy is never touched again.
6. Collateral opens the file, replaces every bad character with a space, and saves the cleaned version over the original location.
7. Collateral returns a short summary: how many cells changed, how many characters were replaced. (**This is nice to have but not a hard requirement**)
8. The owning service carries on as normal parsing, edit checks, validation. It reads the same file it always did, except the file is now clean.
 

## MVP Product Design

 

**1. The cleaning happens at submit time, not at upload time**

Funding calls Collateral when the user submits a funding request. Digital Connect calls Collateral when the user processes a payoff submission. Both call it before the file is parsed or edit-checked. The frontend does not change. It does not call the cleaning service, does not wait for it, and does not need to know it exists.

 

**2. Permission is checked by the owning service, not Collateral**

Funding and Digital Connect already check whether a user is allowed to change a given request. That check happens first, before they call Collateral. Collateral only needs to confirm the call came from a trusted service it does not re-check the user.

 

**3. Collateral reads and writes the blob directly**

Collateral is given permission on the storage container itself, scoped to that one container and not the whole storage account. One permission grant for the funding container, one for the payoff container.

The calling service tells Collateral which container and which file path to work on. This is safe because the caller is another one of our services, not a browser. Collateral still checks the path is inside a container it is configured for, and refuses anything else.

Validate this permission by hand in the Azure portal first. Once copy-and-overwrite works against the funding container, put it into Terraform.

 

**4. The original file is always kept**

Before changing anything, Collateral copies the uploaded file to docs/original/{documentId}/{filename}. That copy is never modified. If anyone ever asks what the user actually sent, or exactly what we changed, this is the answer.

The cleaned file is written over the original location. The document ID, the file name and every message between services stay the same. Nothing downstream needs to be told this happened.

The one exception is .xls files — see requirement 7.

 

**5. The rules live in the database, not in code**

A table holds two lists of allowed characters but we should be able to expand it:

 

**FIELD** — used for most columns:

A-Z  a-z  0-9  space

&  '  ’  @  `  ^  ,  :  {  }  -  $  !  #  (  )  .  |  +  "  ;  /  \  [  ]  *  ~  _  %

 

**DESCRIPTION** — used for description and free-text columns. Everything in FIELD, plus:

<  >  ?  carriage return  tab

The lists are read from the database every time a file is cleaned. They are never hardcoded in Python, Java or the frontend. Changing a list is a database change, not a code release. Each version of the list has a version number so we can tell which rules cleaned which file.

For the MVP, the lists are updated by SQL. A screen for editing them can come later.

 

**6. Each column gets the right list**

Collateral already has a schema describing the columns in these files. It uses that schema to decide which list applies to each column.

If a column is marked as a description column, it gets the DESCRIPTION list. Everything else gets FIELD. If a column is not in the schema at all, it gets FIELD, because FIELD is the stricter of the two.

< and > stay out of the FIELD list until the ProMerit review of account key fields is finished. Travis flagged that those characters should not be used in account key fields until confirmed.

 

**7. What "cleaning" does**

 

For each cell, go through it character by character:

- If the character is on that column's list, keep it.
- If it is not, replace it with a single space.
 

Then squeeze any run of two or more spaces down to one and remove spaces from the start and end.

 

Example: A™™B becomes A B.

Only ordinary spaces are squeezed. Tabs and line breaks are left alone because the DESCRIPTION list allows them.

 

**8. Which files we handle**

 

**CSV** — must be UTF-8. If it is not, reject it with a clear message. Comma-separated. Strip the byte order mark when reading. Write line endings as \n.

.xlsx — edit the text cells in place. Numbers, dates, true/false values, blanks and formulas are left completely alone. Formatting and merged cells are preserved.

.xls — read it, then write the cleaned version out as a .xlsx. This is the one place where the file cannot be replaced in place, because the format changes. The cleaned .xlsx gets a new path and the owning service updates its document record to point at it. The original .xls is still archived.

Rejected: macro-enabled files (.xlsm), funding files with more than one sheet, and anything over 100 MB. Everything runs synchronously.

 

**9. We record what changed, but not the actual values**

 

For each cleaned file we save which service, which document, which rule version, the file's ETag, how many cells changed, how many characters were replaced, which columns were affected, which characters were removed, when it ran, and who triggered it.

We deliberately do **not** save the original cell contents. Those could contain customer data, and storing them would mean encrypting them, controlling who can read them, and deleting them on a schedule — all of which is a lot of work for the MVP. The archived original file already tells us everything we need.

 

**10. Retries, failures and rollout**

 

If the same file is cleaned twice with the same rules, the second call returns the first result instead of doing the work again. We match on service, document ID, rule version and file ETag.

If cleaning fails, the submit or process fails too, with an error the user can retry. We never carry on with an uncleaned file.

The whole thing sits behind a feature flag, CHARACTER_REMEDIATION_ENABLED, which stays off in production until we have verified it.