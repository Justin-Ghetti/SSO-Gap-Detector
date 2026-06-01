# SSO Gap Detector

> Automatically detects Campus-Wide License email domains whose single sign-on (SSO) status is misrecorded, validates the real configuration through live browser-based detection, and routes each gap to the right owner with a formatted email. Built in Python, Snowflake, and Selenium.

---

## Overview

Campus-Wide Licenses can carry multiple email domains, and each one may or may not be configured for SSO. The licensing application was never designed around SSO, and because the status is maintained through manual changes, the stored record drifts out of step with reality. When a domain is marked as not-SSO in the database but a sibling domain on the same license has SSO enabled, that is almost always a gap that will eventually cause a customer login failure.

This tool finds those gaps automatically. It pulls candidate domains from Snowflake, verifies the real SSO configuration in a live browser session, and routes each confirmed gap to the correct action through a formatted email, on a weekly schedule.

---

## The Problem (Before)

The licensing system had no built-in way to verify SSO configuration, so confirming it was a manual, recurring chore:

- SSO status was tracked in an ad-hoc spreadsheet, separate from the system of record
- Verifying domains meant a manual review process, repeated regularly
- Because SSO changes were made by hand, the database fell out of sync with reality easily, and nothing automatically caught it
- The first sign of a problem was often a customer who could not log in

The core gap: a domain shows a not-applicable SSO status in the database, but a sibling domain on the same license has SSO enabled — a strong signal the first domain is misconfigured.

---

## The Solution (After)

A scheduled Python pipeline that:

1. **Extracts** Campus-Wide License domains and their recorded SSO status from Snowflake, grouped so siblings on the same license can be compared
2. **Deduplicates** against a known-domain list so each new gap is only actioned once
3. **Validates** each candidate by driving a live browser session to detect whether SSO redirection actually happens, independent of the database record
4. **Loads** the result by sending a formatted HTML email that routes the correct next action based on what detection found

**No more manual review. No more side-of-desk spreadsheet. Gaps are caught, validated, and routed automatically before they become login failures.**

---

## Stack

| Layer | Tool |
|---|---|
| Data Source | Snowflake (entitlement + audit-trail schemas) |
| Orchestration | Python 3.11+ |
| Validation | Selenium + headless Chrome (live SSO redirect detection) |
| Scheduling | Windows Task Scheduler (portable to cron / Airflow) |

---

## How It Works

> Note: schema names, table names, status values, and the authentication endpoint are described generically here. Internal system details are intentionally omitted.

### Extract
A Snowflake query returns Campus-Wide License email domains, their recorded SSO status, and enough license context to group sibling domains together.

### Deduplicate
A state manager compares the current results against a known-domain list so the pipeline only acts on genuinely new gaps, not ones already handled.

### Validate
For each candidate domain, the pipeline opens a live browser session and checks whether authentication actually redirects to an SSO provider. This is the key step: it establishes ground truth directly, rather than trusting the database record.

### Load / Route
Based on the validation result, the pipeline sends a formatted HTML email that routes the correct action:

| Validation Result | Action |
|---|---|
| SSO detected in reality | Route to update the database record to match |
| SSO not detected | Route to the remediation owner with fix instructions |
| Detection error | Flag for manual review, with debug screenshots saved |

---

## Run Modes

The tool supports a dry-run mode (drafts emails without sending), a visible-browser mode for debugging the detection step, and a limit option to process only a handful of domains during testing. A full run validates all new gaps and sends live emails.

---

## Project Structure

```
sso_gap_detector/
├── main.py            # Orchestrator
├── config.py          # Settings loaded from environment
├── snowflake_conn.py  # Snowflake connection + query runner
├── state_manager.py   # Known-domain list + deduplication
├── sso_checker.py     # Selenium-based SSO detection
├── email_sender.py    # HTML email templates + delivery
├── logs/              # One log file per run (audit trail)
├── screenshots/       # Selenium debug artifacts on failure
└── email_drafts/      # HTML drafts saved in dry-run mode
```

---

## Key Design Decisions

- **Validate reality, don't trust the record.** The whole premise is that the database can be wrong, so the tool confirms SSO through live detection rather than relying on the stored status.
- **Compare siblings, not single domains.** Looking across all domains on a license catches misconfigurations a per-domain check would miss.
- **Deduplicate with a state list.** Tracking known domains means each gap is actioned once, so re-runs do not spam owners.
- **Route by result, not just alert.** Each detection outcome maps to a specific next action and audience, so the email drives resolution instead of just reporting a problem.
- **Built-in audit trail.** Every run logs its results and saves screenshots on failure, so the process is reviewable after the fact.

---

## Roadmap

- Move the in-memory known-domain list to a dedicated Snowflake tracking table
- Auto-create CRM cases on new gaps via REST API
- Bidirectional sync to close tracker rows when the corresponding case resolves

---

## Notes on Sensitive Data

This repository is a sanitized portfolio version. Schema names, table names, status values, the authentication endpoint, internal file paths, and individual names have been removed or genericized. No production query, real credentials, or customer data is included.
