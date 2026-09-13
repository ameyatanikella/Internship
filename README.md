# Tabroom Coach Contact Extractor

A Python utility that converts Tabroom.com tournament-registration pages into a structured, outreach-ready contact list of high school speech and debate coaches.

Built during a summer internship to replace a manual copy-and-paste workflow with a repeatable script.

---

## Problem

Tabroom.com is the registration platform used by the overwhelming majority of U.S. high school speech and debate tournaments, which makes it the single best source of coach contact information for anyone doing school outreach. That information is visible to logged-in users, but it is:

- **Behind a session login**, so a plain HTTP request returns nothing useful
- **Paginated by state**, with no export button and no public API
- **Rendered as nested, class-based HTML**, where a coach's email, name, school, and state each live in a different element

Assembling an outreach list by hand meant opening one state at a time and transcribing hundreds of rows field by field. This tool reduces that to: capture the page, run the script, import the CSV.

## Approach

The script treats a saved page as a line-indexed document and keys off the CSS class markers Tabroom uses for each column, reading the value from a known offset relative to each marker.

| Marker in source | Data extracted | How |
| --- | --- | --- |
| `<span class="tenth">` | State | Two-character abbreviation on the following line |
| `<span class="twofifths">` | School | Line index recorded; each email is attributed to the nearest preceding school header |
| `mailto:` | Email + coach name | Address sliced out of the `href`; name read from the following line |

Because contacts are grouped under a school heading rather than repeating it per row, school attribution is a back-reference to the closest earlier school marker, which keeps every email tied to the right institution.

## Features

- **Authenticated capture with no credential handling.** The optional Selenium mode launches Chrome against an existing user-data directory and profile, reusing a session the user has already signed into. No usernames, passwords, or tokens appear in the code or the repository.
- **Marker-relative parsing.** Fields are located by their class markers and line offsets instead of hardcoded row numbers, so the script tolerates states with different numbers of schools and coaches.
- **School back-referencing.** Each contact is matched to the most recent school heading above it, preserving the page's implicit grouping.
- **Encoding cleanup.** Strips stray non-ASCII artifacts introduced by the saved page source so downstream fields import cleanly.
- **Append-mode output.** Results accumulate across runs, so all 50 state pages can be processed in sequence into one dataset.
- **Trailing-row trim.** Automatically removes Tabroom's own support address, which appears as a footer link on every page.

## Tech stack

- **Python 3** — file I/O and string processing (standard library only for the parsing path)
- **Selenium WebDriver** — optional browser automation for capturing authenticated page source
- **ChromeDriver** — driven against an existing Chrome profile

## Usage

**1. Capture a state page.**

Either save the page source manually from a logged-in browser as `state.txt`, or uncomment the Selenium block at the top of `scrape.py` and point it at your Chrome user-data directory and profile.

**2. Run the script.**

```bash
python scrape.py
```

**3. Use the output.**

Rows are appended to `data.txt` in comma-separated form. Rename to `.csv` and open in any spreadsheet tool or mail-merge system.

Repeat for each state; output accumulates into a single file.

## Output format

```csv
email,coach_name,school,state
coach@example.edu,Jane Doe,Example High School,CA
```

## Design notes

Choices worth calling out, and what a production version would change:

- **Line-based parsing over a DOM library.** Reading the saved source line by line kept the script dependency-free and easy to debug against a static file. The tradeoff is fragility: any change to Tabroom's markup breaks the offsets. A rewrite would use BeautifulSoup with CSS selectors for the same fields.
- **String concatenation over the `csv` module.** Fast to write, but it does not quote or escape values, so a school name containing a comma would shift columns. `csv.writer` would handle this correctly.
- **Hardcoded paths.** Input and output paths are literals; `argparse` would make the script portable across machines.
- **No deduplication.** Coaches listed at multiple schools appear more than once and are currently deduplicated downstream in the spreadsheet.

## Scope and responsible use

The script parses pages the signed-in user is already authorized to view, at the pace of a person browsing manually. It stores no credentials, and the resulting contact data was used solely for the internship's outreach program, not redistributed or resold.
