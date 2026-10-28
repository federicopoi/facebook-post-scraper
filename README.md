# Facebook Post Scraper

Turn a Facebook business-page URL into a local ZIP of its available photos, business details and accessible reviews. Runs directly from the command line with **your own Facebook login cookies**. No Apify account, Facebook developer account or API key is required.

This repository contains only the scraper. It does not include a website builder, anyone’s login session, environment configuration, downloaded photos, or live customer run records.

## Requirements

- Python **3.11 or newer**.
- [uv](https://docs.astral.sh/uv/getting-started/installation/) (recommended), or pip in a virtual environment.
- Your own working Facebook login, supplied by a Netscape-format cookie file or an accessible local browser profile.
- Permission to access and download the target page’s content.

## Quick start: cookie file

```sh
git clone https://github.com/federicopoi/facebook-post-scraper.git
cd facebook-post-scraper
uv run scripts/scrape_to_zip.py 'https://www.facebook.com/PAGE_HANDLE' --cookies '/absolute/path/to/facebook-cookies.txt'
```

`uv` installs the script’s declared dependencies automatically. Replace the example page and cookie-file path with your own. Quote the URL, especially when it contains `?` or `&`.

The command prints a JSON result with the final `zip_path`, image/review counts, source-data warnings, archive verification and cleanup status. The default destination is your account’s `Downloads` directory. `--downloads` chooses another folder.

### Getting your login cookies

1. Sign into **your own** Facebook account in your browser. Resolve any login challenge there first.
2. Either use browser-profile mode below (no export required), or use a trusted local cookie-export tool to export **only `facebook.com` cookies**, in **Netscape cookies.txt format**. A JSON export, copied `Cookie:` header or Facebook password is not accepted.
3. Save that file **outside this repository**, then pass its path with `--cookies`. It must contain valid, unexpired `c_user` and `xs` login cookies. Export all Facebook cookies for best compatibility, rather than manually pasting individual values. HttpOnly cookie rows are supported.
4. If the login expires, sign in again and replace your local export. Cookie presence alone cannot guarantee Facebook will accept the session or expose a particular page.

A cookie file grants access to your account. **Never send it to someone else, commit it, attach it to an issue, paste it into a chat, or upload it to a cookie-conversion website.** If possible, restrict the file to your user account (`chmod 600 /path/to/facebook-cookies.txt` on macOS/Linux). Review any export tool’s permissions before using it. See [SECURITY.md](SECURITY.md).

In cookie-file mode, the script reads only Facebook-domain cookies into memory, excludes expired cookies and unrelated domains, does not open a browser profile, and does not rewrite/export the cookie file. Photo CDN downloads use a separate session without your Facebook cookies.

## Alternative: use your browser login directly

```sh
uv run scripts/scrape_to_zip.py 'FACEBOOK_URL' --browser chrome
uv run scripts/scrape_to_zip.py 'FACEBOOK_URL' --browser edge --profile 'Default'
uv run scripts/scrape_to_zip.py 'FACEBOOK_URL' --browser firefox
```

Chrome is the default browser; there is **no hard-coded personal browser profile**. Omit `--profile` to use the browser reader’s default, or choose your own profile name/path. `--browser` and `--cookies` are alternative authentication sources; do not combine them. `--profile` applies only to browser login.

Browser-cookie extraction depends on your operating system, browser encryption and profile permissions. macOS may ask for Keychain access. Some browser/OS combinations prevent decryption; use your own Netscape cookie export in that case. The scraper does not automate login or bypass session restrictions.

## Without uv

```sh
python -m venv .venv
# macOS/Linux
source .venv/bin/activate
# Windows PowerShell: .venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
python scripts/scrape_to_zip.py 'FACEBOOK_URL' --cookies '/path/to/facebook-cookies.txt'
```

On Windows, `py` may be your Python launcher. Pass a Windows cookie path such as `C:\Users\YOUR_NAME\facebook-cookies.txt` rather than the Unix examples above. Scripts use Python paths; no Bash script is required. Tests are offline and portable; a live run on every OS/browser combination is not guaranteed.

## Options

```sh
# All available photos (default); Facebook can still limit visibility
uv run scripts/scrape_to_zip.py 'FACEBOOK_URL' --cookies '/path/to/facebook-cookies.txt'

# Deliberately request a smaller collection
uv run scripts/scrape_to_zip.py 'FACEBOOK_URL' --cookies '/path/to/facebook-cookies.txt' --limit 20

# Choose output location or archive name
uv run scripts/scrape_to_zip.py 'FACEBOOK_URL' --cookies '/path/to/facebook-cookies.txt' --downloads './output' --name 'Example Business'

uv run scripts/scrape_to_zip.py --help
```

| Option | Default | Meaning |
| --- | --- | --- |
| `--cookies PATH` | Unset | Netscape-format Facebook cookie file; skips browser reading |
| `--browser NAME` | `chrome` | Browser to read when no cookie file is supplied |
| `--profile NAME_OR_PATH` | Browser default | Your local browser profile |
| `--downloads PATH` | `~/Downloads` | Destination for the verified ZIP |
| `--limit N` | `0` | Maximum photos; zero requests all exposed photos |
| `--name TEXT` | Detected business name | Override the ZIP’s business-name prefix |
| `--workers N` | `6` | Concurrent image downloads |
| `--doc-id ID` | Code default | Override the photo query ID after a Facebook schema change |
| `--review-doc-id ID` | Code default | Override the independent review query ID |

Vanity page URLs, numeric `profile.php?id=...` URLs, `/people/name/ID/` URLs and resolvable shared page links are supported. This is a business-page photo collector, not a general private-profile or group scraper.

### Photos and manifest only

The lower-level command keeps the downloaded folder and produces `photo-index.csv` and `report.json`, without business metadata, reviews, ZIP creation or raw-folder cleanup:

```sh
uv run scripts/fb_photos.py 'https://www.facebook.com/profile.php?id=PAGE_ID' --cookies '/path/to/facebook-cookies.txt' --output './output'
```

Use a numeric page URL for this lower-level command; use the main ZIP command for vanity/share-link resolution.

## What is in the ZIP?

Each run creates a unique ZIP with one business folder containing:

- Available photos and an optional profile-picture candidate (not automatically a verified logo).
- `business.md`: readable contact/About details, attributed reviews and coverage warnings.
- `business.json`: structured business fields, sources, relative image paths and availability flags.
- `photo-index.csv`: image filenames, public source links, sizes, dimensions and SHA-256 hashes.
- `report.json`: collection status, pagination and download diagnostics.

The tool validates image decoding, dimensions, byte counts and hashes, then reads and verifies every ZIP member before removing **only that run’s** extracted folder. Existing archives are never overwritten. If a photo download or ZIP check fails, it preserves the working folder and reports failure. If only cleanup fails, a verified ZIP can exist alongside the remaining raw folder; inspect the JSON result.

A verified ZIP does not imply all Facebook fields were available. Missing/blocked contacts and partial reviews remain explicitly marked. A verified empty photo collection can still return business data. [Full bundle schema](references/bundle-schema.md).

## Limitations and troubleshooting

- **Login/permission problems:** restore your Facebook login and refresh your cookie export. Check that you can view the target page in that account. Do not share login material when asking for help.
- **Missing business fields or reviews:** Facebook may not return them to your session. The tool records missing/partial coverage instead of inventing values.
- **Throttling or refused queries:** bounded retries are built in. Stop rather than repeatedly retrying a refused collection.
- **Facebook changes:** this uses Facebook’s internal web responses and persisted GraphQL queries, not a stable public API. Query IDs and response schemas can change. The underlying collection queries were checked in October 2026; future operation is not guaranteed. [Maintenance guide](references/maintenance.md).
- **Photo scope:** collects exposed page photos. It does not download videos, post comments, arbitrary full post timelines, restricted content, or Google reviews. Downloaded dimensions do not prove original-upload quality.
- **Storage:** temporary uncompressed media and the final ZIP both need disk space. A failed run intentionally leaves its recovery folder.
- **Reuse:** respect the content owner’s rights, Facebook’s terms, and the access granted to your account. Downloaded business information and reviews may contain personal data; review a bundle before sharing it.

## Development

```sh
PYTHONPATH=scripts uv run --with pillow --with 'gallery-dl==1.32.17' python -m unittest discover -s tests -v
```

For PowerShell, first set `$env:PYTHONPATH = "scripts"`, then run the same `uv run ...` command without the `PYTHONPATH=scripts` prefix. Tests use synthetic fixtures and do not need login cookies or make live Facebook requests. Keep downloaded media and live-run reports outside the repo. A contribution must not contain secrets or account data.

`AGENTS.md`, `CODEX.md`, `CLAUDE.md` and `SKILL.md` provide optional agent instructions for this scraper only. No website-building tools are bundled.

## License

[MIT](LICENSE). This covers the scraper code, not Facebook content or third-party dependencies.
