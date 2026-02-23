# Module: Shareable Public Links

Adds the ability to generate public, token-based links that allow unauthenticated access to specific data.

---

## Server Additions

### ShareLinksController

- Route: `/api/share-links`
- `POST /` — Create a new share link for a resource (authenticated)
- `GET /` — List share links created by the authenticated user
- `DELETE /{token}` — Revoke a share link (authenticated)

### SharedDataController

- Route: `/api/shared`
- `GET /{token}` `[AllowAnonymous]` — Retrieve the shared data using the public token

### ShareLinkService

Core operations:
- Generate a unique token (GUID or crypto-random string)
- Store link metadata: token, owner user ID, resource identifier, creation date, optional expiry
- Validate token and retrieve associated resource
- Revoke (delete) a share link

Key decisions:
- **Link expiry**: Should links expire after a set duration, or persist until manually revoked?
- **Access control**: Is the shared data read-only, or can recipients interact with it?
- **What data is shared**: Define exactly which resource fields are exposed publicly — never expose internal IDs, user PII, or sensitive fields through share links
- **Rate limiting**: Consider limiting the number of active share links per user

### DynamoDB Key Pattern (if using DynamoDB)

| Entity | PK | SK | GSI1PK | GSI1SK |
|--------|----|----|--------|--------|
| ShareLink | `SHARE_LINK#{token}` | `USER#{userId}` | `USER_SHARES#{userId}` | `SHARE#{createdAt:O}` |

The GSI enables querying all share links for a given user.

---

## Web Additions

Optional pages:
- **ShareLinks.razor** — Manage share links (create, list, revoke)
- **SharedView.razor** `[AllowAnonymous]` — Public page that renders shared data from the token in the URL

---

## File Additions

```
Server/src/Server/
  Controllers/
    ShareLinksController.cs
    SharedDataController.cs
  Services/
    IShareLinkService.cs
    ShareLinkService.cs
  Models/
    ShareLink.cs
```
