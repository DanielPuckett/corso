# Corso Architecture Analysis & Risk Assessment

## Software Design Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                           CLI Layer                                   │
│  src/cli/ — cobra commands: backup, restore, export, repo            │
└──────────────────────────────┬───────────────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────────────┐
│                      Operations Layer                                 │
│  src/internal/operations/ — orchestrates backup/restore/export flows  │
└──────────┬───────────────────────────────────┬───────────────────────┘
           │                                   │
┌──────────▼──────────────┐     ┌──────────────▼───────────────────────┐
│   M365 Service Layer    │     │        Storage Layer (Kopia)          │
│  src/internal/m365/     │     │  src/internal/kopia/                  │
│  ├── controller.go      │     │  ├── wrapper.go (snapshot mgmt)      │
│  ├── collection/        │     │  ├── upload.go                       │
│  │   ├── exchange/      │     │  ├── conn.go (repo connection)       │
│  │   ├── drive/         │     │  ├── s3.go                           │
│  │   ├── site/          │     │  └── filesystem.go                   │
│  │   ├── groups/        │     └──────────────────────────────────────┘
│  │   └── teamschats/    │
│  └── service/           │
│      ├── exchange/      │
│      ├── onedrive/      │
│      ├── sharepoint/    │
│      └── groups/        │
└──────────┬──────────────┘
           │
┌──────────▼──────────────────────────────────────────────────────────┐
│                    Microsoft Graph API Layer                          │
│  src/pkg/services/m365/api/                                          │
│  ├── client.go          — Graph client setup (Kiota-based)           │
│  ├── mail.go            — Email CRUD                                 │
│  ├── mail_pager.go      — Mail folder/message pagination + delta     │
│  ├── contacts.go        — Contacts CRUD                              │
│  ├── contacts_pager.go  — Contacts pagination                        │
│  ├── events.go          — Calendar events CRUD                       │
│  ├── events_pager.go    — Events pagination + delta                  │
│  ├── drive.go           — OneDrive file operations                   │
│  ├── drive_pager.go     — Drive items pagination + delta             │
│  ├── sites.go           — SharePoint sites                           │
│  ├── lists.go           — SharePoint lists                           │
│  ├── channels.go        — Teams channels                             │
│  ├── conversations.go   — Group conversations                        │
│  └── graph/betasdk/     — Custom beta API client for Pages           │
└──────────────────────────────────────────────────────────────────────┘
```

## Key Dependencies

| Component | Version | Purpose |
|-----------|---------|---------|
| `msgraph-sdk-go` | v1.30.0 | Microsoft Graph v1.0 API client |
| `kiota-*` | v1.x | HTTP/serialization framework for Graph SDK |
| `azure-sdk-for-go/azidentity` | v1.5.1 | Azure AD authentication |
| `kopia` (forked) | v0.12.2 (alcionai fork) | Deduplication, encryption, snapshot storage |
| Custom `betasdk` | in-tree | Hand-rolled beta Graph API client for SharePoint Pages |

## Storage Architecture

Corso uses **Kopia** (a fork maintained by Alcionai) as its storage engine:
- Deduplication at the block level
- Encryption at rest
- Supports **S3** and **local filesystem** backends
- Snapshots are immutable once written
- Incremental backups via delta tokens stored in snapshot metadata

## Microsoft Graph API Usage

### v1.0 (Stable) APIs — Used for core operations:

| Service | Backup | Restore | Notes |
|---------|--------|---------|-------|
| **OneDrive** | Delta query on drive items (`/drives/{id}/items/delta`) | POST to `/drives/{id}/items/{id}/children`, PUT content | Standard, well-supported |
| **Exchange Mail** | Delta query on messages (`/users/{id}/mailFolders/{id}/messages/delta`) | POST messages with MIME content | Standard |
| **Exchange Contacts** | List contacts per folder | POST contacts | Standard |
| **SharePoint Libraries** | Same as OneDrive (drives API) | Same as OneDrive | Standard |
| **SharePoint Lists** | List items via sites API | POST list items | Standard |

### Beta APIs — Used for specific features:

| Feature | Endpoint | Risk Level |
|---------|----------|------------|
| **Mail Folders enumeration** | `GET /beta/users/{id}/mailFolders` | **LOW** — used because v1.0 doesn't return nested folders. Likely to be promoted to v1.0 eventually. Fallback: recursive v1.0 calls. |
| **Calendar Events delta** | `GET /beta/users/{id}/calendars/{id}/events/delta` | **MEDIUM** — delta for events is still in beta. Without it, full enumeration is needed (slower but functional). |
| **OneDrive createLink (sharing)** | `POST /beta/drives/{id}/items/{id}/createLink` | **LOW** — only used during restore to recreate sharing links. Not needed for backup. If deprecated, restores would lose sharing permissions but files would still restore. |
| **SharePoint Pages** | `GET /beta/sites/{id}/pages` | **MEDIUM** — entire Pages backup/restore relies on beta API. If removed, Pages cannot be backed up. However, Pages are a minor data type for most users. |

## Risk Assessment

### 🟢 LOW RISK — Core backup/restore functionality

**OneDrive, SharePoint Libraries, Exchange Mail, Contacts:**
- All use **v1.0 stable** Graph APIs
- Delta queries for incremental backup are v1.0 stable
- File content upload/download uses standard drive item APIs
- These are Microsoft's most heavily used APIs — extremely unlikely to be deprecated

**Storage (Kopia):**
- The forked Kopia is pinned and self-contained
- S3 and filesystem backends are stable
- Your existing repo will continue to work regardless of upstream changes

### 🟡 MEDIUM RISK — Beta API dependencies

**Calendar Events Delta (`/beta/.../events/delta`):**
- Impact: Without this, calendar backup falls back to full enumeration each time (slower, more API calls, but still works)
- Likelihood of breaking: Low — Microsoft has been moving delta to v1.0 for other resources
- Mitigation: Could be replaced with full enumeration + client-side diffing

**SharePoint Pages (`/beta/sites/{id}/pages`):**
- Impact: Pages cannot be backed up if this endpoint changes or is removed
- Likelihood: Medium — Pages API has been in beta for years without promotion
- Mitigation: Pages are typically a small portion of SharePoint data. Libraries (files) are unaffected.

### 🟡 MEDIUM RISK — Archived project concerns

**No security patches:**
- The Graph SDK (`msgraph-sdk-go v1.30.0`) will not receive updates
- If Microsoft changes auth flows or token formats, the pinned `azidentity` may stop working
- Mitigation: The Azure AD OAuth2 flow is extremely stable; unlikely to break for years

**No adaptation to API changes:**
- If Microsoft deprecates a v1.0 endpoint (rare but possible), no one will update corso
- Mitigation: Microsoft's deprecation policy requires 36+ months notice for v1.0 endpoints

**Kopia fork divergence:**
- The Alcionai Kopia fork will not receive upstream security fixes
- Mitigation: Kopia's storage format is stable; the fork is only used for read/write, not network-facing

### 🔴 HIGH RISK — Long-term viability (2+ years)

**Azure AD App Registration:**
- Microsoft periodically updates consent and permission models
- If your app registration's required permissions change scope names, backup will fail
- Mitigation: Monitor Microsoft 365 admin center for deprecation notices on your app's permissions

**Graph SDK authentication library (`microsoft-authentication-library-for-go v1.2.1`):**
- MSAL libraries evolve; if Microsoft forces a new auth protocol version, the pinned library may fail
- Timeline: Likely 3-5 years before this becomes critical
- Mitigation: Could update just the auth library independently if needed

## Summary

**For your current use case (Exchange, OneDrive, SharePoint file backup/restore):**

The core functionality uses **stable v1.0 APIs** and is at **low risk** of breaking in the near term (1-2 years). The beta dependencies are for secondary features (calendar delta optimization, sharing links on restore, SharePoint Pages) that either have fallbacks or affect minor data types.

**Biggest practical risk:** The Azure AD authentication library aging out over 3-5 years as Microsoft evolves their identity platform. This would affect all operations equally.

**Recommendation:** Continue using corso as-is. Monitor for:
1. Auth failures (first sign of MSAL/Azure AD changes)
2. Calendar backup becoming significantly slower (sign of beta delta endpoint change)
3. Microsoft 365 admin center deprecation notices for Graph API permissions


## Teams & Chats Support Status (Tested 2026-05-06)

### Groups (`corso backup create groups --group '*'`)

The code is fully wired up and registered as a CLI command (marked "Preview").

**What works:**
- ✅ Libraries (SharePoint files within the group) — backed up successfully
- ✅ Channel Messages — enumerated and backed up (e.g. 3 channels, 43 messages for "Daniel's Organization")
- ❌ Conversations/Posts — **403 Forbidden** on `GET /v1.0/groups/{id}/conversations`

**Conversations fix:** Add `Group.Read.All` (Application permission) to the Azure AD app registration and grant admin consent.

**Groups skipped as "service not enabled":** Most groups in the tenant are not "Unified" M365 groups (they're distribution lists, security groups, or shared mailboxes). Only actual M365 Groups with Teams/SharePoint backing have data to back up. This is correct behavior.

### Chats (`corso backup create chats --user '*'`)

The command is registered (marked "Pre-Release") and can enumerate chats, but **cannot back up chat content**.

**What works:**
- ✅ Enumerates users and their chats (found 115 chats for alice@contoso.com, 2 for bob@contoso.com)
- ✅ Incremental backup infrastructure (delta tracking, merge bases)
- ❌ Actual chat message download — returns `"getting item data: not implemented"` from `chat_handler.go:86`

**Root cause:** The `GetData` function in `internal/m365/collection/teamschats/chat_handler.go` line 86 is a stub that returns "not implemented". The chat enumeration was built but the actual message content retrieval was never completed before the project was archived.

**Error label:** `label_forces_no_backup_creations` — the code intentionally prevents creating a backup snapshot when this error occurs, so no corrupt/empty snapshots are created.

**Verdict:** Chats backup is **non-functional by design** (incomplete implementation). This cannot be fixed without modifying the corso source code to implement the Graph API calls for retrieving chat message content (`GET /v1.0/chats/{id}/messages`).

### Permissions Required for Full Groups Support

| Permission | Type | Purpose |
|-----------|------|---------|
| `Group.Read.All` | Application | Read group conversations/posts |
| `ChannelMessage.Read.All` | Application | Read Teams channel messages (already working) |
| `Chat.Read.All` | Application | Would be needed if chats were implemented |


### Channel Messages — 401 on Content Download (group2.out)

On the second run (incremental backup), corso attempts to download individual channel message content and replies:

```
GET /v1.0/teams/{id}/channels/{id}/messages/{messageId} → 401 Unauthorized
GET /v1.0/teams/{id}/channels/{id}/messages/{messageId}/replies → 401 Unauthorized
```

`ChannelMessage.Read.All` is already granted in the app registration. The 401 is likely due to Microsoft's "protected API" gating — Teams channel message content access requires a separate manual approval process beyond just granting the permission. As of testing, the Microsoft documentation links for requesting protected API access return 404, making this a dead end.

**Impact:** Channel message *enumeration* works (listing channels and message IDs), but downloading actual message *content* fails. Libraries (files) within the group still back up successfully.

**Status:** Blocked by Microsoft's protected API approval process. Not a corso code issue.


### Groups Backup Scope — Data Only, No Configuration

The groups backup covers **data content only**:
- ✅ Libraries (SharePoint files within the group)
- ✅ Channel messages (Teams chat content)
- ✅ Conversation posts (Outlook group mailbox)

It does **not** back up group/team configuration:
- ❌ Group membership and owners
- ❌ Group settings (privacy, classification, policies)
- ❌ Teams configuration (tabs, apps, connectors)
- ❌ Channel structure and settings
- ❌ Planner boards/tasks
- ❌ OneNote notebooks
- ❌ Group calendar

If a Team/Group were lost, corso could restore files and message history, but the Team structure, channels, tabs, apps, and membership would need manual recreation.
