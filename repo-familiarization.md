# cabrillolog: repo familiarization notes

Status: first pass, September 2026. These notes record what this library does, what it depends on, and who uses it. They do not propose fixes. Items that look like defects are listed as observations for later triage.

Sources: this fork (`kawfey/cabrillolog`, `main` at `ff0b747`, identical to `N0SO/cabrillolog` `main`) and read-only clones of Mike N0SO's other MOQP repos. For the wider MOQP picture, see `repo-familiarization.md` in `kawfey/moqputils`.

No code was run. Every module imports `cabrilloutils`, which is not in any public repo.

## Contents

1. [Summary](#summary)
2. [Repo map](#repo-map)
3. [Classes](#classes)
4. [Field mapping to the MOQP database](#field-mapping-to-the-moqp-database)
5. [Dependencies](#dependencies)
6. [Who uses this library](#who-uses-this-library)
7. [History](#history)
8. [Observations noticed in passing](#observations-noticed-in-passing)
9. [Open questions](#open-questions)

## Summary

- cabrillolog is a small object model (about 870 lines of Python) for MOQP Cabrillo logs: a header object, a QSO object and a log-file object that holds both.
- One object can load from a Cabrillo file or from a row of the MOQP MySQL database (`LOGHEADER`, `QSOS`), and it can give its data back in either form. The translation between the two forms is by list position in `headerdefs.py`.
- It is a newer, object-based rewrite of the parsing that moqputils does in `moqplogfile.py` and `moqpqsoutils.py`. moqputils does not import it.
- Its only confirmed user is `N0SO/reverselog`.
- It depends on two private packages that are not in any public repo: `cabrilloutils` (all modules) and `multicounty` (`logfile.py`). The database path also needs `moqputils`.
- The QSO parser is specific to the 10-field MOQP exchange. It does not validate field values.

## Repo map

```
cabrillolog/
├── __init__.py      empty
├── headerdefs.py    tag lists: Cabrillo header tags and QSO fields, each beside its DB column list
├── cabheader.py     class cabrilloHeader
├── qso.py           class QSO
├── logfile.py       class logFile (header + list of QSO)
├── cbltest          manual test script (prints a parse of AA0AJ.LOG)
├── README.md        one-line description
└── LICENSE
```

## Classes

### `cabrilloHeader` (`cabheader.py`)

- One attribute per Cabrillo 3 header tag, named with `_` in place of `-` (for example `CATEGORY_POWER`). There are also three extra attributes that exist only in the database: `ID`, `CABBONUS` and `TIMESTAMP`.
- `parseHeader(x)`: if `x` is a dict (a `LOGHEADER` row), it calls `parseHeaderdb`. Otherwise it treats `x` as a list of header lines. It splits each line at the first `:`, changes the text to upper case, and stores it with `setTagData`. It prints a message for lines without a `:` and for unknown tags.
- `setTagData(tag, data)`: maps `-` to `_`. If the tag is `END-OF-LOG`, it stores `True`.
- Output: `makeTSV()` and `makeHTML()` give one row of 29 fields. `prettyprint('csv'|'html')` gives `TAG: value` lines or a two-column HTML table. `getHeader()` returns `vars(self)`.

### `QSO` (`qso.py`)

- Attributes: `freq`, `mode`, `qtime` (a `datetime`), `mycall`, `myrst`, `myqth`, `urcall`, `urrst`, `urqth`, plus the database fields `valid`, `dupe`, `note`, `id`, `logid`, `qsl`, `nolog`, `noqsos`. `dbdata` is `True` when the QSO came from the database.
- `parseQSO(x)`: if `x` is a dict (a `QSOS` row), it calls `parseQSOdb`. Otherwise it removes a leading `QSO:`, splits on white space and fills the fields in order. The date and time fields go to `cabrilloutils.qsoutils.QSOUtils.qsotimeOBJ` and become `qtime`. If the line does not have exactly 10 fields, the parser adds a message to `qerrors`. If it has fewer than 10, it also sets `valid=False`.
- `getQSO()` returns `vars(self)`. `getDBQSO()` returns a dict keyed by `QSOS` column names.
- Output: `makeTSV()`, `MakeidTSV()` (with ID, valid and dupe), `makeHTML()`. The database form prints more columns (QSL, NOLOG and NOQSOS).

### `logFile` (`logfile.py`)

- Holds `header` (a `cabrilloHeader`), `qsoList` (a list of `QSO`), `fileName`, `rawlog` and `nextQID`.
- The first constructor argument decides the source:

| Argument | Source |
|---|---|
| path of an existing file | `getLogFromFile`: read the file (`QSOUtils.readFile`), then `parseRawLog` |
| string shorter than 10 characters that is not a file | taken as a callsign: `getLogfromDB` reads `LOGHEADER` and `QSOS` through `moqputils.moqpdbutils.MOQPDBUtils` |
| string that starts with `START-OF-LOG:` | intended for raw log text (see observations) |
| list of lines | `parseRawLog` |

- `parseRawLog`: `extractHeader` takes the lines from `START-OF-LOG:` up to the first `QSO:` line. `extractQSOS` takes every `QSO:` line and passes it through `multicounty.multiCounty`. That class splits a county-line QSO (county written as `ct1/ct2`) into one QSO per county (added in 2023). If every QSO has a valid `datetime`, the QSOs are sorted by time. Each QSO then gets an `id` from 0 upward in list order.
- `PrettyPrint()` returns the header lines followed by one TSV line per QSO.

## Field mapping to the MOQP database

`headerdefs.py` has two pairs of lists. In each pair, item *n* of one list maps to item *n* of the other. The comments in the file say that a change to the order breaks the database methods.

| Header attribute | `LOGHEADER` column | | QSO attribute | `QSOS` column |
|---|---|---|---|---|
| START_OF_LOG | START | | freq | FREQ |
| CALLSIGN | CALLSIGN | | mode | MODE |
| OPERATORS | OPERATORS | | qtime | DATETIME |
| CONTEST | CONTEST | | mycall | MYCALL |
| LOCATION | LOCATION | | myrst | MYREPORT |
| CATEGORY_BAND / _MODE / _OPERATOR / _POWER / _STATION | CATBAND / CATMODE / CATOPERATOR / CATPOWER / CATSTATION | | myqth | MYQTH |
| CATEGORY_TRANSMITTER / _OVERLAY / _ASSISTED | CATXMITTER / CATOVERLAY / CATASSISTED | | urcall | URCALL |
| CATEGORY_TIME | (not in DB) | | urrst | URREPORT |
| CERTIFICATE, CLUB | CERTIFICATE, CLUB | | urqth | URQTH |
| CLAIMED_SCORE, CREATED_BY | CLAIMEDSCORE, CREATEDBY | | valid, dupe, note | VALID, DUPE, NOTE |
| EMAIL, NAME, ADDRESS | EMAIL, NAME, ADDRESS | | id, logid | ID, LOGID |
| ADDRESS_CITY / _STATE_PROVINCE / _POSTALCODE / _COUNTRY | CITY / STATEPROV / ZIPCODE / COUNTRY | | qsl, nolog, noqsos | QSL, NOLOG, NOQSOS |
| OFFTIME, SOAPBOX | OFFTIME, SOAPBOX | | | |
| GRID_LOCATOR | (not in DB) | | | |
| END_OF_LOG | ENDOFLOG | | | |
| ID, CABBONUS, TIMESTAMP | ID, CABBONUS, TIMESTAMP | | | |

`LOGHEADER` has columns that the header object does not map: `IOTAISLANDNAME`, `MOQPCAT` and `STATUS` (from the 2020 sample schema in moqputils). The `QSOS` mapping uses `DATETIME`, so it matches the current moqputils loader, not the 2020 sample schema (`DATE` and `TIME` strings).

## Dependencies

| Package | Used in | Used for | Available? |
|---|---|---|---|
| `cabrilloutils.qsoutils.QSOUtils` | all modules | `qsotimeOBJ` (date/time parse), `readFile` | **no**, private (probably `/home/pi/Projects/` on the Pi) |
| `multicounty.multicounty.multiCounty` | `logfile.extractQSOS` | expand county-line QSOs (`isMulti()`, `qsoList`, `qsoText`) | **no**, private |
| `moqputils.moqpdbutils`, `moqputils.configs.moqpdbconfig` | `logFile.getLogfromDB` | database read | yes, `kawfey/moqputils` (needs a local DB config) |
| `headerdefs`, `qso`, `cabheader` | inside this repo | | yes |

Import style: the modules import each other with bare names (`from qso import QSO`, `from headerdefs import ...`). Code outside the repo imports them as `cabrillolog.logfile`. Both forms work only when `sys.path` contains the parent folder (for example `/home/pi/Projects`) and the `cabrillolog` folder itself. That is why the MOQP scripts add `/home/pi/Projects/cabrillolog` to `sys.path`.

## Who uses this library

Search of the public N0SO repos and `kawfey/moqputils`:

| Repo | Use |
|---|---|
| `N0SO/reverselog` | `reverseLog` is a subclass of `logFile`. It builds a "reverse" Cabrillo log for a callsign from other stations' QSOs in the database, and uses `QSO.parseQSO`. |
| `N0SO/rookiereport`, `N0SO/status1x1` | Their entry scripts put `/home/pi/Projects/cabrillolog` on `sys.path`, but no module imports from cabrillolog (the path list looks copied from another script). |
| `kawfey/moqputils` | No imports. It has its own parser. |
| Pi log report (`/moqp/logtools/moqp2026/pylogreport.php`) | Not confirmed. The 2023 commit "Updates adding logreport support" and upstream moqputils issue #64 (`mqplogreport` rejected 222 MHz and microwave QSOs) point to a log report tool that uses this library, but that tool is in no public repo and the Pi was not reachable. |

## History

- 16 commits, all by Mike N0SO, from 2022-07-13 to 2024-05-17. The fork is identical to upstream. There are no branches other than `main`, and no issues are referenced in commits.
- 2022-07 (13 commits): first version. The file and database read/write were added over two weeks and ended with "Beta version ready for testing!" (2022-07-31).
- 2023-04: county-line expansion (`ct1/ct2`) through `multicounty`.
- 2023-05: "adding logreport support" (changes to `qso.py`, `logfile.py` and `cabheader.py`).
- 2024-05: "updates for 2024" (`qso.py` output formats, header fields).

## Observations noticed in passing

These were found while reading the code. Nothing here was confirmed by running it.

1. `logFile.__init__` for a raw-text string calls `fileName.upper.startswith(...)`. `upper` has no `()`, so this path raises `AttributeError`.
2. `logFile.__init__` for a list of lines stores the result in `self.qsolist` (lower case `l`), so `self.qsoList` stays empty for that path. The `self.rawlog` test on that branch also cannot be true there.
3. `__testRawlog` reads `self.RAWLOG` when no argument is given. Only `getLogFromFile` sets that attribute, so a direct call without an argument would fail.
4. `parseHeader` keeps only the last line of a tag that appears more than once. Cabrillo allows repeated `ADDRESS:`, `SOAPBOX:` and `OPERATORS:` lines. The field-by-field approach also means that `X-` tags and any unknown tags are only printed as warnings.
5. `parseQSO` does not validate values (its docstring says "Needs error checking added!"). A QSO parsed from a file keeps the default `valid=False`. It returns `False` for a complete line, because `index > MAXELEMENTS` can never be true. No caller seen uses the return value.
6. A QSO line with more than 10 fields (for example a transmitter ID or serial number) is parsed with the extra fields ignored, and a message goes to `qerrors`.
7. `cbltest` is a manual script, not a unit test. It needs `AA0AJ.LOG` (not in the repo) and has hard-coded Pi and Windows paths.
8. There is no packaging (`setup.py`/`pyproject.toml`) and no `requirements.txt`.

## Open questions

1. Where are `cabrilloutils` and `multicounty` kept, and can they be added to the kawfey org as repos?
2. Is the Pi's log report tool (`mqplogreport` / `pylogreport.php`) built on this library, and where is its source?
3. Should cabrillolog replace the parsing in moqputils (`moqplogfile.py`, `moqpqsoutils.getQSOdict`), or should it stay a separate library for the side tools?
4. Should the QSO parser stay specific to the MOQP exchange, or should it take the field layout per contest as Cabrillo intends?
