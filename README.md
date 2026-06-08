# waybackwhen

A multi-source passive URL enumerator that aggregates historical endpoints from the Wayback Machine, Common Crawl, AlienVault OTX, URLScan, VirusTotal, and more — all in one run.

## What It Does

Runs five passive tools against a domain and merges the results into a single deduplicated output file. Useful for surface mapping, endpoint discovery, and parameter hunting without touching the target directly.

**Sources covered:**
| Tool | Sources |
|------|---------|
| waybackurls | Wayback Machine |
| gau | Wayback Machine, Common Crawl, AlienVault OTX, URLScan |
| waymore | Wayback Machine, URLScan, VirusTotal, Common Crawl |
| urlfinder | Wayback Machine, Common Crawl |
| paramspider | Wayback Machine (parameter-focused) |

**Tool parallelism:** `gau` and `urlfinder` run in parallel (different backends). `waybackurls`, `waymore`, and `paramspider` run sequentially to avoid hammering the Wayback CDX API simultaneously.

---

## Installation

### Dependencies

Install the Go tools:

```bash
go install github.com/tomnomnom/waybackurls@latest
go install github.com/lc/gau/v2/cmd/gau@latest
go install github.com/projectdiscovery/urlfinder/cmd/urlfinder@latest
```

Install waymore:

```bash
git clone https://github.com/xnl-h4ck3r/waymore.git ~/tools/waymore
pip install -r ~/tools/waymore/requirements.txt
```

Install paramspider:

```bash
pip install paramspider
```

Install tldextract (required for `--apex` flag):

```bash
pip install tldextract
```

### Install waybackwhen

```bash
git clone https://github.com/DFC302/waybackwhen.git
cd waybackwhen
chmod +x waybackwhen
sudo cp waybackwhen /usr/local/bin/   # optional: add to PATH
```

### Verify your setup

```bash
waybackwhen --check
```

This prints a status table showing which tools are installed and where:

```
 waybackwhen — tool check
 TOOL            STATUS     PATH
 -----------------------------------------------
  waybackurls    [OK]       /home/user/go/bin/waybackurls
  gau            [OK]       /home/user/go/bin/gau
  urlfinder      [OK]       /home/user/go/bin/urlfinder
  paramspider    [OK]       /usr/local/bin/paramspider
  waymore        [OK]       /home/user/tools/waymore/waymore.py
  python3        [OK]       /usr/bin/python3
  tldextract     [OK]       /usr/lib/python3/dist-packages/tldextract
```

Missing tools are silently skipped at runtime — the script will not crash if a tool is absent, it just won't contribute results.

---

## Usage

```
waybackwhen [options] [domain]
```

### Options

| Flag | Long form | Description |
|------|-----------|-------------|
| `-e` | `--exclude` | Filter out static assets (images, fonts, CSS, JS libraries, etc.) from all results |
| `-a` | `--apex` | Strip subdomain and run all tools against the registered apex domain instead |
| `-c` | `--check` | Check which tools are installed and where, then exit |
| `-s TOOLS` | `--skip TOOLS` | Comma-separated list of tools to skip (e.g. `waymore` or `gau,waybackurls`) |
| `-f FILE` | `--file FILE` | Read domains from a file (one per line) |
| `-p N` | `--parallel N` | Number of domains to process concurrently (default: 1) |
| `-l FILE` | `--log FILE` | Write a timestamped run log to FILE (`.log` extension added if missing) |

**Valid tool names for `--skip`:** `waybackurls`, `gau`, `urlfinder`, `waymore`, `paramspider`

### Input methods

**Single domain:**
```bash
waybackwhen example.com
```

**From file:**
```bash
waybackwhen -f domains.txt
```

**From stdin:**
```bash
cat domains.txt | waybackwhen
```

---

## Examples

**Basic run on a single domain:**
```bash
waybackwhen example.com
```

**Run on a subdomain but pull apex-level history:**
```bash
waybackwhen --apex api.example.com
# Strips to example.com and runs all tools against it
```

**Exclude static assets (cleaner output for endpoint hunting):**
```bash
waybackwhen --exclude example.com
```

**Multiple domains in parallel with logging:**
```bash
waybackwhen -f domains.txt -p 5 -l run.log
```

**Skip a slow tool for a faster run:**
```bash
waybackwhen --skip waymore example.com
```

**Skip multiple tools:**
```bash
waybackwhen --skip gau,waybackurls example.com
```

**Full combination — apex, exclude, skip, parallel, log:**
```bash
waybackwhen -f subdomains.txt --apex --exclude --skip waymore -p 3 -l hunt.log
```

---

## Output

Each domain produces a `.wbw` file in the current directory named after the domain (dots replaced with underscores):

```
example_com.wbw
api_example_com.wbw
```

Files contain one URL per line, sorted and deduplicated. If a domain returns zero results the output file is deleted automatically.

**Example output:**
```
https://example.com/api/v1/users
https://example.com/login?redirect=/dashboard
https://example.com/search?q=FUZZ
https://example.com/wp-login.php
```

---

## Notes

- **Apex flag and multi-part TLDs:** `--apex` uses `tldextract` which handles complex TLDs correctly (`api.example.co.uk` → `example.co.uk`). Falls back to a two-label split if `tldextract` is not installed.
- **paramspider output:** paramspider replaces parameter values with `FUZZ` placeholders (e.g., `?id=FUZZ`). This is intentional — it surfaces parameter-bearing endpoints cleanly.
- **Rate limiting:** waybackurls, waymore, and paramspider all query the Wayback CDX API sequentially by design. Running many domains in parallel (`-p`) multiplies this — use a reasonable concurrency value.
- **waymore config:** waymore returns significantly more results when configured with API keys for URLScan and VirusTotal. See [waymore's README](https://github.com/xnl-h4ck3r/waymore) for setup.
