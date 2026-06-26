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

**Rate-limit resilience:** every tool runs through a retry/backoff wrapper, domains that come back empty are requeued (a common symptom of being throttled), and an optional archive-lane semaphore caps how many domains hit the Wayback/archive tools at once even when overall concurrency is high. See [Rate limiting & resilience](#rate-limiting--resilience).

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
| `-x` | `--exact` | Disable automatic apex extraction and scan the literal input domain (e.g. only `api.foo.com`, not `foo.com`) |
| `-c` | `--check` | Check which tools are installed and where, then exit |
| `-s TOOLS` | `--skip TOOLS` | Comma-separated list of tools to skip (e.g. `waymore` or `gau,waybackurls`) |
| `-f FILE` | `--file FILE` | Read domains from a file (one per line) |
| `-p N` | `--parallel N` | Number of domains to process concurrently (default: 1) |
| `-l FILE` | `--log FILE` | Write a timestamped run log to FILE (`.log` extension added if missing) |
| | `--stdout` | Print merged URLs to stdout instead of writing `.wbw` files (status/progress goes to stderr, so output stays pipe-clean) |
| `-r N` | `--retries N` | Per-tool retries on a hard failure (non-zero exit), with exponential backoff (default: 2) |
| `-q N` | `--requeue N` | Whole-domain retries when a pass returns zero URLs, with backoff (default: 1). Catches silent throttling, where a tool exits cleanly but returns nothing |
| `-b N` | `--backoff N` | Base backoff in seconds; the delay for attempt _n_ is `N × n` (default: 5) |
| `-a N` | `--archive-slots N` | Max domains allowed to hit the archive lane (`waybackurls`/`waymore`/`paramspider`) at once. `0` = unlimited (default). Clamped to `--parallel`. Lets you fan out wide with `-p` without stampeding the Wayback API |

**Valid tool names for `--skip`:** `waybackurls`, `gau`, `urlfinder`, `waymore`, `paramspider`

All numeric flags accept non-negative integers; `--parallel` must be at least `1`.

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

**Run on a subdomain (apex is extracted automatically by default):**
```bash
waybackwhen api.example.com
# Strips to example.com and runs all tools against it.
# Prints: [*] Apex extracted: api.example.com → example.com
```

**Scan the literal subdomain only (no apex strip):**
```bash
waybackwhen --exact api.example.com
# Scans api.example.com directly, does NOT widen to example.com.
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

**Full combination — exclude, skip, parallel, log (apex is the default):**
```bash
waybackwhen -f subdomains.txt --exclude --skip waymore -p 3 -l hunt.log
```

**Pipe URLs straight into another tool (stdout mode):**
```bash
waybackwhen --stdout example.com | httpx -silent
```

**Fan out wide but stay gentle on the Wayback API:**
```bash
# 20 domains in flight, but only 4 hitting the archive tools at any moment
waybackwhen -f domains.txt -p 20 -a 4
```

**Recommended rate-limit-safe profile for large lists:**
```bash
waybackwhen -f domains.txt -p 20 -a 4 -q 1 -b 8 -l run.log
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

## Rate limiting & resilience

Running many domains in parallel (`-p`) used to mean up to that many `waybackurls` + `waymore` processes all hammering `web.archive.org` at once, with no recovery if any of them got throttled. Three layers address that:

1. **Per-tool retry with backoff (`-r`, default 2).** Each tool runs through a wrapper that re-runs it on a hard failure (non-zero exit), waiting `backoff × attempt` seconds between tries. This catches crashes and network errors.

2. **Whole-domain requeue on empty (`-q`, default 1).** The archive tools usually exit `0` even when they were rate-limited, so retry-on-error alone misses silent throttling. When a domain's merged result is empty, the entire tool battery is re-run after a backoff. A genuinely empty domain just costs one extra (bounded) backoff cycle before its output is dropped.

3. **Archive-lane semaphore (`-a`, default unlimited).** A counting semaphore caps how many domains hit the archive tools (`waybackurls`/`waymore`/`paramspider`) simultaneously, while `gau` and `urlfinder` keep running at full `-p`. This is the cleanest way to prevent throttling in the first place: e.g. `-p 20 -a 4` fans out to 20 domains but lets only 4 touch the archives at a time. The value is clamped to `--parallel`.

Tune the backoff base with `-b` (seconds). Set `-q 0` to disable requeuing, or `-a 0` for the old unlimited behavior.

## Notes

- **Default apex extraction and multi-part TLDs:** apex extraction (the default) uses `tldextract`, which handles complex TLDs correctly (`api.example.co.uk` → `example.co.uk`). Falls back to a two-label split if `tldextract` is not installed. Use `--exact` / `-x` to bypass and scan the literal input.
- **paramspider output:** paramspider replaces parameter values with `FUZZ` placeholders (e.g., `?id=FUZZ`). This is intentional — it surfaces parameter-bearing endpoints cleanly.
- **waymore config:** waymore returns significantly more results when configured with API keys for URLScan and VirusTotal. See [waymore's README](https://github.com/xnl-h4ck3r/waymore) for setup.
