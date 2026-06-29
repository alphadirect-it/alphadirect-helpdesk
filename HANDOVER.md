# Alpha Direct IT Help Desk — Handover & Admin Runbook

**Version:** 1.0
**Date:** 2026-06-29
**Outgoing owner:** Ikanyeng Sechele (IT@alphadirect.co.bw)
**Incoming owner:** Prathap Ganesharajah (CFO)
**Status:** Live, single-tenant Alpha Direct Insurance

---

## 1. System Overview

A single-page web application that lets Alpha Direct staff log IT support tickets, watched and worked by IT staff, with all data persisted in SharePoint and notifications driven by Power Automate.

**Live URL (current, temporary):** https://alphadirect-it.github.io/alphadirect-helpdesk/
**Target URL (pending DNS cutover via Arjun):** https://helpdesk.alphadirect.co.bw

**Stack:**
- Frontend: single HTML file (vanilla JS + MSAL.js v2.38.0) — served via GitHub Pages
- Authentication: Microsoft Entra ID (Azure AD) — Single-page application flow with PKCE
- Backend storage: SharePoint Online list
- Automation: Power Automate flows for email notifications
- No servers, no databases, no client secrets

---

## 2. Architecture

```
[Browser]
   |
   v  HTTPS
[GitHub Pages: alphadirect-it.github.io/alphadirect-helpdesk]
   |
   v  MSAL.js (Auth Code + PKCE)
[Microsoft Entra ID]
   |
   v  Access token (scopes: User.Read, Sites.ReadWrite.All, GroupMember.Read.All)
[Microsoft Graph API]
   |
   v  Reads / writes
[SharePoint: alphadirectbw.sharepoint.com/sites/ITHelpDesk → list "IT Tickets"]
   |
   v  Item created / modified events
[Power Automate flows]
   |
   v  SMTP via Office 365 Outlook connection
[Email recipients: IT@alphadirect.co.bw, requesters, assignees]
```

The frontend never holds credentials. All auth is delegated to Microsoft Entra ID; tokens are managed in browser session storage by MSAL.

---

## 3. Code & GitHub

**Current repo:** github.com/alphadirect-it/alphadirect-helpdesk (Public)
**Branch:** `main`
**Deploy target:** GitHub Pages → `/(root)` of main branch
**Build:** none. Static HTML.

**Files in repo:**
- `index.html` — the entire app (HTML + CSS + JavaScript in one file, ~3000 lines)
- `README.md` — auto-generated placeholder
- `HANDOVER.md` — this document

**To update the live site:** edit `index.html` directly on GitHub (or upload a new version). GitHub Actions rebuilds and redeploys in ~1–2 minutes. No manual deploy steps.

**No GitHub Actions workflows defined.** GitHub Pages uses its built-in Jekyll-bypass build (because the file is at root and there's no Jekyll config).

---

## 4. Microsoft Entra ID (Azure AD)

**App registration name:** Alpha Direct IT Help Desk
**Application (client) ID:** `e0a3b758-f709-4bf0-a4aa-f855458a5ee1`
**Directory (tenant) ID:** `fb4aec07-793a-494c-92f6-7a527a7f89c0`
**Supported account types:** Single tenant (Alpha Direct only)
**Platform:** Single-page application (SPA)

### Current Redirect URIs (SPA)

- `http://localhost:8080` — local development (deprecated, can remove)
- `http://localhost:5500` — local development with Python http.server
- `http://localhost:5500/helpdesk-v2.html` — local development specific file
- `https://alphadirect-it.github.io/alphadirect-helpdesk/` — production via GitHub Pages
- *To add after DNS cutover:* `https://helpdesk.alphadirect.co.bw/`

### API Permissions (all Delegated, admin-consented)

- `User.Read` (Microsoft Graph) — sign-in and basic profile
- `Sites.ReadWrite.All` (Microsoft Graph) — read/write the SharePoint list
- `GroupMember.Read.All` (Microsoft Graph) — read membership of IT-HelpDesk-Agents / IT-HelpDesk-Admins for role detection

### Authentication flow

OAuth 2.0 Authorization Code Flow with PKCE. **There is no client secret.** SPAs use PKCE in place of a client secret — confidential credentials cannot be stored safely in browser code, so the PKCE challenge proves possession without one. No secret expiry to track.

### Implicit grant and hybrid flows: BOTH UNCHECKED

---

## 5. SharePoint Backend

**Site:** alphadirectbw.sharepoint.com/sites/ITHelpDesk
**Site type:** Private team site
**List name:** `IT Tickets`

### List schema (13 columns + system columns)

| Column | Type | Required | Notes |
|---|---|---|---|
| Title | Single line of text | Yes | Short summary of the issue |
| TicketID | Single line of text | Yes | Auto-generated `TKT-####` by the web app |
| Description | Multiple lines of text | Yes | Plain text, no rich text |
| Category | Choice | Yes | Hardware, Software, Network, Access / Account, Email & Communication, Printing, Mobile / BYOD, Other |
| Priority | Choice | Yes | Low, Medium, High, Critical (default Medium) |
| Status | Choice | Yes | Open, In Progress, On Hold, Resolved, Closed (default Open) |
| Department | Choice | Yes | Finance, Underwriting, Claims, Data Analytics, IT, HR, Sales, Operations, EXCO, Other |
| Requester | Person | Yes | Set by the web app from M365 sign-in identity |
| AssignedTo | Person | No | Set by Admin/Agent in the modal |
| FirstResponseAt | Date and time | No | Stamped by web app when status → In Progress |
| ResolvedAt | Date and time | No | Stamped by web app when status → Resolved |
| Comments | Multiple lines of text | No | Append-only activity log; each entry has `[timestamp] author:\ntext` |
| SLAHours | Calculated | (auto) | Formula: `=IF([Priority]="Critical",4,IF([Priority]="High",8,IF([Priority]="Medium",24,IF([Priority]="Low",72,24))))` |
| BreachNotified | Yes/No | No | Set to Yes by Flow 4 once SLA breach alert has been sent |

### List views

- **My Tickets** — filter: Requester = `[Me]`; sort: Created desc
- **Open Queue** — filter: Status ≠ Resolved AND ≠ Closed; sort: Priority then Created
- **My Assigned** — filter: AssignedTo = `[Me]`; sort: Created desc
- **SLA At Risk** — filter: Status ≠ Resolved AND ≠ Closed AND Created < [Today]-1
- **All Tickets** — no filter; sort: Created desc

### Permissions

- Item-level: **Read items created by user; Create and edit items created by user** (under List Settings → Advanced settings)
- "Everyone except external users" → **Contribute** (for end-user ticket submission)
- IT Help Desk Members (Azure AD group `IT-HelpDesk-Agents`) → **Edit**
- IT Help Desk Owners (Azure AD group `IT-HelpDesk-Admins`) → **Full Control**

---

## 6. Azure AD Groups (Role Control)

| Group | Purpose | Members |
|---|---|---|
| **IT-HelpDesk-Agents** | Detected by the web app to grant Agent role. SharePoint Edit access. Assignable in dropdown. | IT technicians |
| **IT-HelpDesk-Admins** | Detected by the web app to grant Admin role. SharePoint Full Control. Sees Dashboard + reports. | IT Manager, CIO |

To grant Agent or Admin access to someone: add them as a **Member** (not Owner) of the relevant group in Entra. Propagates within 60 seconds.

---

## 7. Power Automate Flows

All flows owned by Ikanyeng's account currently. **Recommend re-pointing to a shared IT service account** before final handover so ownership survives staff changes.

### Flow 1 — IT Helpdesk — Form to Ticket *(legacy, can be deprecated)*

- **Trigger:** Microsoft Form submission ("Log an IT Support Ticket")
- **Purpose:** Translates Microsoft Forms submissions into SharePoint items. Originally created when the front-end was a Form. The web app at GitHub Pages now creates SharePoint items directly, so this flow is redundant unless the Form is kept as a backup channel.
- **Status:** Active. Safe to disable once the web app is fully adopted.

### Flow 2 — IT Helpdesk — New Ticket Notifications

- **Trigger:** SharePoint "When an item is created" on `IT Tickets`
- **Purpose:** Sends two emails on every new ticket:
  1. To `IT@alphadirect.co.bw` — IT inbox notification (HTML body)
  2. To Requester's email — confirmation with ticket number
- **Connections:** SharePoint, Office 365 Outlook
- **Notes:** Uses regular `Send an email (V2)` (not shared mailbox version) because `IT@alphadirect.co.bw` is on-premise Exchange. `Reply To` header is set to `IT@alphadirect.co.bw` so replies route back to IT inbox.

### Flow 3 — IT Helpdesk — Resolution Notification

- **Trigger:** SharePoint "When an item is created or modified" on `IT Tickets`
- **Trigger conditions:**
  - `@equals(triggerOutputs()?['body/Status/Value'], 'Resolved')`
  - `@empty(triggerOutputs()?['body/ResolvedAt'])`
- **Purpose:** When status flips to Resolved (and ResolvedAt is empty, to prevent re-fire loops), update the ResolvedAt timestamp and email the requester asking for confirmation.
- **Loop protection:** the second trigger condition ensures the flow only fires once per resolution event.

### Flow 4 — IT Helpdesk — SLA Breach Check

- **Trigger:** Scheduled (every 1 hour, Africa/Gaborone timezone)
- **Purpose:** Scans `IT Tickets` for items where Status ≠ Resolved/Closed AND elapsed hours > SLAHours AND BreachNotified = No. For each match, email IT Manager and flip BreachNotified to Yes.
- **Filter Query on Get items:** `Status/Value ne 'Resolved' and Status/Value ne 'Closed' and BreachNotified eq 0`

### Flow 5 — IT Helpdesk — Closed Notification

- **Trigger:** SharePoint "When an item is created or modified" on `IT Tickets`
- **Trigger condition:** `@equals(triggerOutputs()?['body/Status/Value'], 'Closed')`
- **Purpose:** When a ticket is closed, email two parties:
  1. Requester — closure notice
  2. Assignee (if any) — closure confirmation for their records
- **Conditional logic:** Inside the flow, a Condition step checks `empty(triggerOutputs()?['body/AssignedTo/Email'])`. If the ticket had no assignee, only the requester email fires.

---

## 8. Common Operations

### Adding a new IT technician (Agent)

1. Entra admin centre → Groups → IT-HelpDesk-Agents
2. Members tab → + Add members → search and select the person
3. Wait 60 seconds for propagation
4. Person can now sign into the help desk and see the Agent view; they appear in the Assignee dropdown for everyone

### Updating the help desk code

1. Edit `index.html` on GitHub (web UI, or clone+push)
2. Commit to `main`
3. GitHub Actions automatically deploys (no manual step)
4. Within 1–2 minutes, the live site reflects changes
5. Users hard-refresh (Ctrl+Shift+R) to bypass browser cache

### Adding a new redirect URI to Azure AD

When deploying to a new domain (e.g. `helpdesk.alphadirect.co.bw`):
1. Entra → App registrations → Alpha Direct IT Help Desk → Authentication
2. Under Single-page application, click + Add URI
3. Paste the new URL (must match what MSAL sends, including trailing slash)
4. Save

### Cleaning up old test tickets

Items with TicketID like `tk0014` (lowercase, wrong format) are leftover from early development. Safe to delete from SharePoint manually. They do not affect production behaviour.

---

## 9. Known Quirks

- **Pagination:** Microsoft Graph caps list responses at 100 items per page. The web app paginates via `@odata.nextLink` (up to 50 pages = 5000 items). If the list grows beyond that, raise the `MAX_PAGES` constant in `index.html`.
- **Person fields:** Graph returns Person columns only as `RequesterLookupId` (an integer). The web app maintains an in-memory cache of all SharePoint users (queried from the hidden `User Information List`) to resolve names. If the User Information List query fails, names show as `User #N`.
- **On-premise mailbox:** `IT@alphadirect.co.bw` is on-prem Exchange. The original Power Automate setup attempted to use the "Send from a shared mailbox" action which requires cloud-hosted mailboxes. Current flows use regular Send Email V2 + Reply-To header.
- **No client secret:** This is a Single Page Application. Browser-side code cannot safely hold a secret; PKCE replaces it. Nothing to rotate.

---

## 10. Secrets / Configuration Inventory

| Item | Where it lives | Sensitive? |
|---|---|---|
| Azure AD client ID | Hardcoded in `index.html` line ~480 | No — public identifier |
| Azure AD tenant ID | Hardcoded in `index.html` line ~480 | No — public identifier |
| SharePoint site URL | Hardcoded in `index.html` | No |
| SharePoint list name | Hardcoded in `index.html` | No |
| Azure AD group names | Hardcoded in `index.html` | No |
| **Client secret** | N/A — SPA uses PKCE | N/A |
| **Connection strings / API keys** | N/A — all auth via MSAL token flow | N/A |

**Nothing to hand over via Vault or 1Password. There are no secrets in this system.** All authorisation is governed by Entra group membership and SharePoint permissions, both of which Prathap will inherit through the access steps below.

---

## 11. Outstanding / Future Work

- **Custom domain cutover:** `helpdesk.alphadirect.co.bw` — DNS sits with Arjun (Cloudflare). After cutover, GitHub Pages "Enforce HTTPS" should be enabled and the new URL added as a redirect URI in Entra.
- **Service account ownership:** Re-point Power Automate flows from Ikanyeng's personal account to a shared IT service account so ownership survives staff changes.
- **Power BI dashboards:** Visual reporting (trends, agent performance, category distribution) is a natural next step. SharePoint list connects directly.
- **Mobile responsiveness:** The web app works on mobile but isn't optimised. Could be improved.
- **Decommission Flow 1 + Microsoft Form:** Once the web app is the only ticket-creation channel, the legacy Form-to-Ticket flow and the Form itself can be retired.

---

## 12. Contacts

- **System author:** Ikanyeng Sechele, IT, IT@alphadirect.co.bw
- **Incoming owner:** Prathap Ganesharajah, CFO, pganesharajah@alphadirect.co.bw
- **DNS for cutover:** Arjun (Cloudflare admin)
- **Microsoft tenant:** alphadirect.co.bw / alphadirectbw.sharepoint.com

---

*End of handover document.*
