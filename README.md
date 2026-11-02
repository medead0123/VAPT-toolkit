# VAPT Toolkit - Automated Web App Penetration Test & Report Generator

A custom-built Python toolkit that automates reconnaissance, vulnerability scanning, and
professional report generation for a web application penetration test, demonstrated against
OWASP Juice Shop (a deliberately vulnerable app used for legal security training).

This project covers both halves of a real VAPT engagement:
1. **Automated scanning** - port scan, tech fingerprinting, security header/cookie/TLS checks,
   optional OWASP ZAP active scan.
2. **Manual testing + reporting** - a schema for recording manually-found vulnerabilities
   (SQLi, XSS, IDOR, auth bypass, etc.), merged with automated findings into one CVSS-scored,
   OWASP/CWE-mapped PDF report.

---

## 1. Prerequisites (Windows 11)

Install these once:

1. **Python 3.11+**: https://www.python.org/downloads/ - during install, check
   **"Add python.exe to PATH"**.
2. **VS Code**: https://code.visualstudio.com/ - then install the **Python** extension
   (Extensions icon in the left sidebar -> search "Python" -> Install, by Microsoft).
3. **Docker Desktop**: https://www.docker.com/products/docker-desktop/ - needed to run the
   Juice Shop lab target (and optionally OWASP ZAP). Start Docker Desktop and let it finish
   loading before continuing.
4. **nmap** (optional but recommended for richer port/service scan results):
   https://nmap.org/download.html - Windows installer. If you skip this, the toolkit
   automatically falls back to a built-in Python socket scanner.

Verify in a terminal (PowerShell):
```powershell
python --version
docker --version
```

---

## 2. Get the project into VS Code

1. Create a folder, e.g. `C:\Users\<you>\Projects\vapt-toolkit`.
2. Copy all the project files (the ones provided) into that folder, preserving the folder
   structure shown below.
3. In VS Code: **File -> Open Folder...** -> select `vapt-toolkit`.
4. Open a terminal inside VS Code: **Terminal -> New Terminal** (it opens PowerShell by default
   in that folder).

Expected structure:
```
vapt-toolkit/
  README.md
  requirements.txt
  config.yaml
  docker-compose.yml
  vapt/
    cli.py
    models.py
    utils.py
    recon/
      port_scanner.py
      tech_fingerprint.py
    scanner/
      headers_check.py
      cookie_check.py
      tls_check.py
      zap_scan.py
    report/
      generator.py
      templates/report_template.html
  findings/
    sample_findings.json
  docs/
    scope.md
    rules_of_engagement.md
  tests/
    test_models.py
  reports/        (generated reports land here)
  evidence/        (put your Burp screenshots / PoCs here)
```

---

## 3. Set up the Python environment

In the VS Code terminal, from the `vapt-toolkit` folder:

```powershell
python -m venv venv
venv\Scripts\activate
pip install -r requirements.txt
```

You should now see `(venv)` at the start of your terminal prompt. **Every time you reopen VS
Code to work on this project, run `venv\Scripts\activate` again first.**

> **Note on WeasyPrint (PDF export):** WeasyPrint needs the GTK runtime on Windows to render
> PDFs. If `pip install` succeeds but PDF generation later fails with a DLL/library error, you
> have two options:
> - Install the GTK3 runtime: https://github.com/tschoonj/GTK-for-Windows-Runtime-Project/releases
>   (download and run the installer, then restart VS Code), or
> - Just use the generated HTML report (`reports/pentest_report.html`) - open it in Chrome/Edge
>   and press **Ctrl+P -> Save as PDF**. The toolkit automatically falls back to this and tells
>   you if PDF export fails, so the project still works end-to-end either way.

---

## 4. Start the lab target (OWASP Juice Shop)

Still in the VS Code terminal:

```powershell
docker compose up -d
```

Wait ~15-20 seconds, then open http://localhost:3000 in your browser - you should see the Juice
Shop storefront. This is your authorized, local-only test target.

(This also starts an OWASP ZAP daemon on port 8080 if you want to enable the optional ZAP scan
later - see Section 7.)

To stop the lab later: `docker compose down` (add `-v` to fully reset Juice Shop's state).

---

## 5. Run the automated scan

```powershell
python -m vapt.cli scan
```

This will:
- Port-scan `localhost` (nmap if installed, otherwise the built-in socket scanner)
- Fingerprint the tech stack and probe common exposed paths
- Check security headers, cookie flags, and TLS configuration
- Save every automated finding to `findings/findings.json`

You'll see colored progress output in the terminal (via `rich`), ending with a severity summary.

---

## 6. Do the manual testing (this is the core pentest work)

This is where you actually test like a pentester, using the target's exposed attack surface.
Juice Shop has a public list of ~100 known challenges - treat 8-10 of them as your test cases
(SQL injection login bypass, reflected/DOM XSS, IDOR on the basket API, JWT tampering, etc.).

For each vulnerability you confirm:
1. Reproduce it with Burp Suite or curl/Postman and capture the request/response.
2. Take a screenshot and save it under `evidence/`.
3. Record it in the toolkit:

```powershell
python -m vapt.cli add-finding
```

This prompts you interactively for title, description, impact, remediation, CVSS score,
CWE ID, OWASP category, affected URL, and evidence - and appends it to the same
`findings/findings.json` used by the automated scan.

**To get started quickly with realistic example content**, you can copy the provided sample
findings (based on Juice Shop's well-known vulnerability classes) in as a starting point, then
edit them to match your own actual test evidence/screenshots:

```powershell
copy findings\sample_findings.json findings\findings.json
```

> Treat the sample file as a template showing the level of detail expected, not as something to
> submit unedited - replace the evidence with your own captured requests/responses and
> screenshots once you've reproduced each issue yourself.

---

## 7. (Optional) Run an OWASP ZAP active scan

1. Make sure `docker compose up -d` is running (it also starts the `zap` container).
2. Open `config.yaml` and set:
   ```yaml
   scan:
     use_zap: true
   ```
3. Re-run `python -m vapt.cli scan`. This will drive ZAP's spider + active scanner against the
   target and merge any alerts into `findings/findings.json`. This can take several minutes.

---

## 8. Generate the professional PDF report

```powershell
python -m vapt.cli report
```

This produces:
- `reports/pentest_report.pdf` (or `.html` as a fallback - see the WeasyPrint note above)
- `reports/risk_matrix.png` (severity bar chart embedded in the report)

Open the PDF - you'll get a cover page, executive summary with severity breakdown chart, scope
and methodology, a findings index table, one detailed write-up per finding (description,
impact, evidence, remediation, CVSS/CWE/OWASP mapping), and an appendix.

---

## 9. Remediation + retest (what makes this "industry grade")

1. Pick 2-3 of your findings and actually fix them. Easiest examples against Juice Shop's own
   source if you fork it: add the missing security headers (helmet.js), fix cookie flags, or
   patch a specific query. (If you're not patching Juice Shop's source directly, you can instead
   demonstrate the fix conceptually with a code snippet and a retest against your own
   mini test harness - note this clearly in the report.)
2. Re-run the relevant check (`python -m vapt.cli scan`, or retest manually) and confirm the
   finding no longer reproduces.
3. Update that finding's `status` field in `findings/findings.json` to `"Remediated"`, add a
   short retest note to its `evidence` field, and regenerate the report.

---

## 10. Run the tests

```powershell
pytest
```

This checks the CVSS-to-severity mapping and the findings persistence logic.

---

## 11. Project checklist (matches a 4-5 day plan)

- [ ] Day 1: Docker lab running, scope.md and rules_of_engagement.md filled in, `scan` run once
- [ ] Day 2: 8-10 manually confirmed vulnerabilities logged via `add-finding`, evidence captured
- [ ] Day 3: Automated toolkit fully working end-to-end (this repo)
- [ ] Day 4: Report reviewed for quality, risk matrix/exec summary read well
- [ ] Day 5: 2-3 findings remediated and retested, README/demo video recorded, pushed to GitHub

---

## Legal note

This toolkit is built to scan **only systems you own or are explicitly authorized to test**.
OWASP Juice Shop is designed for this purpose. Do not point `config.yaml`'s `target.url` at
any system you don't control.
