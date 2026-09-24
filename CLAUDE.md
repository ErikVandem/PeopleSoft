# CLAUDE.md — PeopleSoft / PeopleCode

Guidance for Claude Code when working in this repository.

## Environment

<!-- TODO: fill in the real values for your installation. -->
- **PeopleTools version:** 8.xx (e.g. 8.60)
- **Application / version:** e.g. HCM 9.2, FSCM 9.2, Campus Solutions 9.2
- **PUM image / update level:** e.g. PI 45
- **Database platform:** Oracle / SQL Server / DB2
- **Customization prefix:** e.g. `WB_` (all custom objects start with this prefix)

When writing code, only use PeopleCode built-ins, classes and meta-SQL that exist in the PeopleTools version above. If unsure whether a function exists in this version, say so rather than guessing.

## Repository layout

PeopleCode lives in the PeopleTools database, not in files. This repo holds **text exports** of it so it can be read, diffed and reviewed. Changes made here must still be migrated into the database via Application Designer / Change Assistant.

```
peoplecode/
  record/             RECORD.FIELD.Event.pcode        e.g. JOB.EMPLID.FieldChange.pcode
  component/          COMPONENT.MARKET.Event.pcode     component-level events
                      COMPONENT.MARKET.RECORD.Event.pcode         component record
                      COMPONENT.MARKET.RECORD.FIELD.Event.pcode   component record field
  page/               PAGE.Activate.pcode
  app-package/        PACKAGE/SUBPACKAGE/Class.pcode  one file per class
  app-engine/         PROGRAM/SECTION.STEP.Action.pcode (PeopleCode actions)
                      PROGRAM/SECTION.STEP.Action.sql   (SQL actions)
  component-interface/ CI_NAME.Method.pcode
  message/            MESSAGE.Event.pcode / service operation handlers
  menu/               MENU.BAR.ITEM.ItemSelected.pcode
sql/                  SQL objects and view definitions (SQLID.sql)
metadata/             Record/field definitions, translate values, etc. (text/CSV/XML)
projects/             Application Designer project exports (XML), untouched
docs/                 Design notes, functional specs
```

Rules for exported files:
- One file per PeopleCode program. The file name encodes the full object path, so never rename without renaming the object.
- Delivered (vanilla) code that has been customized keeps the delivered file name; mark custom blocks inside it (see Customization rules).

## PeopleCode conventions

- **Declare everything.** Use `Local`, `Component` or `Global` with explicit types (`Local Rowset &rs;`, `Local number &i;`). Avoid undeclared variables and `Global` unless there is a clear need.
- **Variables** are prefixed `&`, camel-case after the prefix (`&rsJob`, `&recPersonal`, `&sEmplid` optional Hungarian prefix if the team uses it).
- **Prefer the object model over bare field references** in anything non-trivial: `GetLevel0()`, `GetRowset()`, `GetRow()`, `GetRecord()`, `GetField()`. Be explicit about which buffer level you are on.
- **SQL:**
  - Use bind variables (`:1`, `:2`) — never concatenate user input into SQL strings.
  - Use meta-SQL for portability: `%Table()`, `%DateIn()`, `%DateOut()`, `%CurrentDateIn`, `%Bind()`, `%EffDtCheck()`, `%Substring()`, `%Concat`.
  - Prefer `SQLExec` for single-row lookups, `CreateSQL` / `Fetch` for loops, and SQL objects (`GetSQL(SQL.X)`) for reusable statements.
  - Always handle effective-dating (`EFFDT`, `EFFSEQ`, `EFF_STATUS`) on effective-dated records.
- **Application Packages:** put reusable logic in custom app classes rather than copying code between events. Keep event PeopleCode thin — it should mostly call into app classes.
- **Messages:** use the Message Catalog (`MsgGet`, `MsgGetText`, `Error MsgGet(...)`) rather than hard-coded strings.
- **Error handling:** use `try` / `catch Exception &ex` around integration, file I/O and CI calls; log with `&ex.ToString()`.
- Comments: `/* ... */` for block comments, `REM` / `rem` is acceptable for single lines. Keep comment style consistent with the surrounding file.

## Event model reminders

Pick the right event; getting this wrong is the most common PeopleCode bug.

| Need | Event |
|---|---|
| Default a value | FieldDefault / RowInit |
| React to a user edit | FieldChange |
| Validate a single field | FieldEdit |
| Validate before save (can stop save) | SaveEdit |
| Change data just before DB write | SavePreChange |
| Trigger downstream work after DB write (no UI changes) | SavePostChange |
| Setup when the page/component loads | PostBuild, PreBuild, RowInit, Page Activate |
| Search page behaviour | SearchInit, SearchSave |

- `Error` in FieldEdit/SaveEdit stops processing; `Warning` does not.
- Do not issue `DoSave`, `WinMessage`/`MessageBox` with think-time, or `Transfer` in SavePreChange/SavePostChange/Workflow events.
- Remember RowInit runs for every row loaded into the buffer — keep it cheap.

## Customization rules

- **Never modify delivered objects if a clone, event mapping, or app-class override will do.** Prefer, in order: Event Mapping (PT 8.55+) > Page and Field Configurator > drop-zones > cloned custom object > modifying delivered code.
- If delivered code must be modified, wrap each change:
  ```
  /* WB_CUSTOM BEGIN - <ticket/ref> - <initials> - <date> - <reason> */
  ...
  /* WB_CUSTOM END - <ticket/ref> */
  ```
  <!-- TODO: replace WB_CUSTOM with your team's marker. -->
- All new objects use the customization prefix.

## Working with Claude in this repo

- When asked to write or change PeopleCode, state which **object and event** it belongs in.
- When code depends on record/field names, check `metadata/` first; if the definition isn't there, list the assumed names so they can be verified.
- Code here cannot be compiled or run locally. Flag anything that should be checked in Application Designer (syntax check / "Validate") or tested in a dev environment.
- Don't invent PeopleBooks references or function signatures. If uncertain, say so.
