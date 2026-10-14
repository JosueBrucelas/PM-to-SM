# Password Manager to Secrets Manager Migration Script

`pm_to_sm.py` converts secrets stored in Bitwarden Password Manager into a file that can be imported into Bitwarden Secrets Manager.

Many teams keep API keys, tokens and other machine secrets in Password Manager, either as secure notes or as login items, organized into collections. This script turns those collections into Secrets Manager projects and the items into secrets, then produces a `.json` file that follows Bitwarden's [Secrets Manager import format](https://bitwarden.com/help/import-secrets-data/).

## How data is converted

The script converts one item type per run, chosen with `--item-type`. In both modes, each collection becomes a project with the same name.

**Secure notes** (`--item-type note`, the default)

| Password Manager | Secrets Manager |
|---|---|
| Secure note name | Secret key |
| Secure note body | Secret value |
| Secure note custom fields | Secret note, as `name: value` lines |

**Login items** (`--item-type login`)

| Password Manager | Secrets Manager |
|---|---|
| Login name | Secret key |
| Login password | Secret value |
| Login notes | Secret note |

A login's username, URIs, TOTP seed, custom fields and password history are not carried over.

For example, a collection called **Hunter Exam** containing login items named **Test-SM1**, **Test-SM2** and **Test-SM3** becomes a project called **Hunter Exam** holding three secrets with those names, each using the login's password as its value.

Items of other types (cards, identities, SSH keys, and whichever of notes or logins isn't selected) are ignored. If an organization stores secrets both ways, run the script once for each type and import both files.

## Requirements

- Python 3.7 or newer. No extra packages are needed.
- An **unencrypted** `.json` export of the organization vault from Password Manager. Encrypted exports are rejected.
- An owner or admin account in the organization with access to Secrets Manager, to perform the import.

## Quick start

1. Export the organization vault from the Password Manager Admin Console as unencrypted `.json`.
2. Preview the conversion. Nothing is written in this step.
   ```
   python3 pm_to_sm.py your_export.json --dry-run
   ```
   For login items, add `--item-type login` to this and the next command.
3. Review the summary and any warnings, then generate the import file.
   ```
   python3 pm_to_sm.py your_export.json -o sm_import.json
   ```
4. In Secrets Manager, go to **Settings → Import data**, choose `sm_import.json`, select **Import data**, and enter your master password.
5. Check the new projects and secrets in Secrets Manager.
6. Delete both `sm_import.json` and the original export. Both contain plaintext secrets.

On Windows, use `python` or `py` instead of `python3`.

## Options

| Option | What it does |
|---|---|
| `-o FILE`, `--output FILE` | Output file name. Default is `sm_import.json`. |
| `--item-type note\|login` | Which items to convert: secure notes (default) or login items. |
| `--dry-run` | Show the summary and validation results without writing a file. |
| `--collections NAME [NAME ...]` | Only convert the named collections, e.g. `--collections "SM" "Hunter Exam"`. |
| `--exclude-collections NAME [NAME ...]` | Convert every collection except the named ones, e.g. `--exclude-collections "Finance" "HR"`. Can't be combined with `--collections`. |
| `--include-unassigned` | Also convert items that aren't in any collection. They're imported as secrets with no project. |
| `--keep-empty` | Convert items even when the value would be empty (a secure note with no body, or a login with no password). |
| `--no-fields` | Don't copy custom fields into the secret's note. Applies to secure notes only. |

## Choosing which collections to convert

A full organization export usually contains collections whose items aren't machine secrets. A secure note called "Bank" in a "Finance" collection, or the logins people use every day, shouldn't end up in Secrets Manager. This matters most in login mode, since most collections contain logins. There are two ways to leave those collections out.

**Name the collections to include.** This is best when only a few collections hold secrets.
```
python3 pm_to_sm.py your_export.json --collections "SM" "Hunter Exam"
```
Only the named collections are converted. Every other item is skipped, and `--include-unassigned` is ignored.

**Name the collections to exclude.** This is best when most collections hold secrets.
```
python3 pm_to_sm.py your_export.json --exclude-collections "Finance" "HR"
```
Every collection except the named ones is converted. An item in an excluded collection is always skipped, even if it's also in a collection being converted, and even with `--include-unassigned`.

Collection names must match exactly, including capitalization. If a name isn't found in the export, the script warns you, which helps catch typos. Run with `--dry-run` first to confirm the right notes are included.

Use one of these options rather than editing the export file by hand. Deleting entries from the JSON is easy to get wrong, and a single leftover comma makes the file unreadable.

## What gets skipped by default

The script prints a warning for every item it skips, so nothing is left out silently.

- **Items with no value.** A secure note with no body, or a login with no password, has nothing to use as the secret value.
- **Notes outside the exported collections.** Items can reference collections that aren't included in the export, typically because the exporting user can't see them. Without a collection name there's no project name to use. Have someone with access to all collections run the export to avoid this.
- **Notes with no collection**, unless `--include-unassigned` is used.

## Validation and duplicate checks

Before writing the output file, the script validates it against Secrets Manager's import rules. It checks that:

- every project and secret has a non-empty ID in the correct format, and no ID is used twice;
- every secret's project reference points to a project in the file;
- no secret belongs to more than one project;
- every secret has a key, value and note field.

If any check fails, the errors are listed and **no file is written**.

The IDs in the file are placeholders (`00000000-0000-0000-0000-000000000001`, `...002`, and so on). Bitwarden replaces them with real IDs during import.

The script also handles duplicates within the export:

- **Exact duplicates** (same name, value and notes in the same collection) are dropped, keeping one copy.
- **Same name with a different value** in the same collection keeps both and prints a warning so you can rename one.
- **Two collections with the same name** produce two projects with that name, with a warning.

## Limitations

- **Existing Secrets Manager data isn't checked.** The script only reads the export file. Bitwarden's import doesn't detect existing items either, so importing the same file twice, or importing secrets that already exist, creates duplicates.
- **One project per secret.** Secrets Manager currently allows a secret in only one project. If an item is in several collections, it's placed in the first one and a warning is shown.
- **Machine accounts and access tokens** can't be imported into Secrets Manager. Set these up manually after the import.
- **Attachments** aren't included in Password Manager exports and won't be migrated.

## Security notes

- The console output shows project names and secret keys only, never secret values, so it's safe to share for troubleshooting.
- The export file and the generated import file both contain plaintext secrets. Store them somewhere secure while working and delete them as soon as the import is confirmed.
- Consider deleting or archiving the original items in Password Manager after migrating, so secrets aren't maintained in two places.
