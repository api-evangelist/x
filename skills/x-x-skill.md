---
name: X
description: Use when building applications that interact with X (formerly Twitter) data and functionality. Reach for this skill when agents need to search posts, manage user accounts, stream real-time data, publish content, manage direct messages, or analyze trends through REST API endpoints.
metadata:
    mintlify-proj: x
    version: "1.0"
---

# X API Skill

## Product summary

The X API provides programmatic access to X's public conversation through modern REST endpoints. Agents use it to search posts, retrieve user data, publish content, manage lists and direct messages, stream real-time posts, and analyze trends. The API uses pay-per-usage pricing with no subscriptions. Key files: Bearer Token (from Developer Console), OAuth credentials for user-context requests. Primary CLI: cURL or official SDKs (Python XDK, TypeScript XDK). Base URL: `https://api.x.com/2/`. Authentication: Bearer Token (app-only) or OAuth 1.0a/2.0 (user-context). [Full documentation](https://docs.x.com/x-api/introduction)

## When to use

Reach for this skill when:
- **Searching posts**: Find posts by keywords, hashtags, users, dates, or custom operators (recent 7 days or full archive)
- **Retrieving user data**: Look up user profiles, followers, following lists, or authenticated user info
- **Publishing content**: Create posts, replies, quotes, or manage post edits
- **Managing interactions**: Like, repost, bookmark, or hide replies on posts
- **Streaming real-time data**: Set up filtered streams with rules to receive matching posts as they're published
- **Managing accounts**: Follow/unfollow users, mute, block, or manage lists
- **Direct messaging**: Send, retrieve, or delete direct messages
- **Analyzing data**: Get post counts, trends, or engagement metrics
- **Handling compliance**: Track deleted posts, edited posts, or withheld content

Do NOT use this skill for: UI/UX design, account creation/signup flows, dashboard-only operations, or authentication setup beyond credential retrieval.

## Quick reference

### Authentication methods

| Method | Use case | Credentials |
|:-------|:---------|:------------|
| Bearer Token (app-only) | Read-only, public data | API Key + API Secret → Bearer Token |
| OAuth 1.0a User Context | Post on behalf of user, manage private data | API Key, API Secret, Access Token, Access Token Secret |
| OAuth 2.0 Authorization Code | User-authorized access with scopes | Client ID, Client Secret, Authorization Code |

### Essential endpoints

| Resource | Method | Endpoint | Purpose |
|:---------|:-------|:---------|:--------|
| Posts | GET | `/2/tweets/search/recent` | Search last 7 days |
| Posts | GET | `/2/tweets/search/all` | Search full archive (pay-per-use only) |
| Posts | POST | `/2/tweets` | Create/publish post |
| Posts | DELETE | `/2/tweets/:id` | Delete post |
| Users | GET | `/2/users/by/username/:username` | Look up user by handle |
| Users | GET | `/2/users/:id` | Look up user by ID |
| Stream | GET | `/2/tweets/search/stream` | Connect to filtered stream |
| Stream | POST | `/2/tweets/search/stream/rules` | Add/update stream rules |
| Likes | POST | `/2/users/:id/likes` | Like a post |
| DMs | POST | `/2/dm_conversations/with/:participant_id/messages` | Send direct message |

### Query operators (search)

| Operator | Example | Matches |
|:---------|:---------|:---------|
| Keyword | `python` | Posts containing word |
| Phrase | `"machine learning"` | Exact phrase |
| Hashtag | `#AI` | Posts with hashtag |
| `from:` | `from:xdevelopers` | Posts by user |
| `lang:` | `lang:en` | Posts in language |
| `has:images` | `has:images` | Posts with images |
| `-is:retweet` | `-is:retweet` | Exclude retweets |
| Boolean | `(AI OR ML) -spam` | AND, OR, NOT logic |

### Response structure

All responses follow this pattern:
```json
{
  "data": { /* primary object(s) */ },
  "includes": { /* related objects (users, media, etc.) */ },
  "meta": { /* pagination, result count */ }
}
```

By default, only `id` and `text` fields return. Use `fields` and `expansions` parameters to request additional data.

### Rate limit headers

Check these headers in every response:
- `x-rate-limit-limit`: Max requests in window
- `x-rate-limit-remaining`: Requests left
- `x-rate-limit-reset`: Unix timestamp when window resets

## Decision guidance

### When to use search vs. stream

| Scenario | Use search | Use stream |
|:---------|:-----------|:-----------|
| Historical data | ✓ | — |
| Real-time monitoring | — | ✓ |
| One-time lookup | ✓ | — |
| Continuous listening | — | ✓ |
| Full archive needed | ✓ (pay-per-use) | — |
| Last 7 days | ✓ | ✓ |

### When to use fields vs. expansions

| Need | Use |
|:-----|:-----|
| Additional fields on primary object (post, user) | `fields` parameter (e.g., `tweet.fields=created_at,public_metrics`) |
| Related objects (author, media, polls) | `expansions` parameter (e.g., `expansions=author_id,attachments.media_keys`) |
| Both | Combine: `expansions=author_id&user.fields=username,name` |

### Bearer Token vs. OAuth

| Requirement | Bearer Token | OAuth 1.0a | OAuth 2.0 |
|:------------|:-------------|:-----------|:----------|
| Read public data | ✓ | ✓ | ✓ |
| Post on behalf of user | — | ✓ | ✓ |
| Access private metrics | — | ✓ | ✓ |
| Manage user's DMs | — | ✓ | ✓ |
| Simplicity | ✓ | — | — |

## Workflow

### Typical task: Search and analyze posts

1. **Prepare credentials**
   - Retrieve Bearer Token from Developer Console (Keys and tokens tab)
   - Store securely; never commit to version control

2. **Build query**
   - Start simple: `python` or `#AI`
   - Add operators: `(AI OR ML) lang:en -is:retweet`
   - Test with recent search first (7-day window)
   - Refine based on results

3. **Make request**
   - Choose endpoint: `/2/tweets/search/recent` (all developers) or `/2/tweets/search/all` (pay-per-use)
   - Add fields/expansions: `?tweet.fields=created_at,public_metrics&expansions=author_id&user.fields=username`
   - Set `max_results` (10–100 for recent, 10–500 for archive)
   - Include Bearer Token in Authorization header

4. **Handle response**
   - Check `data` array for posts
   - Match related objects in `includes` by ID
   - Use `meta.next_token` for pagination
   - Parse `public_metrics` for engagement data

5. **Paginate if needed**
   - Save `next_token` from response
   - Pass as `pagination_token` parameter in next request
   - Stop when `next_token` is absent

6. **Verify results**
   - Spot-check posts match query intent
   - Confirm fields are present (not null)
   - Monitor rate limit headers to avoid 429 errors

### Typical task: Create and publish a post

1. **Prepare content**
   - Compose text (max 280 characters, or more with media)
   - Upload media if needed (images, videos, GIFs)
   - Get media IDs from upload response

2. **Build request body**
   ```json
   {
     "text": "Your post text here",
     "media": { "media_ids": ["media_id_1"] },
     "reply": { "in_reply_to_tweet_id": "parent_id" }
   }
   ```

3. **Authenticate with user context**
   - Use OAuth 1.0a or OAuth 2.0 user token (not Bearer Token)
   - Include signature (OAuth 1.0a) or Authorization header (OAuth 2.0)

4. **POST to `/2/tweets`**
   - Response includes `data.id` (post ID) and `data.text`
   - Check for errors in `errors` array

5. **Verify**
   - Confirm post ID returned
   - Check post appears on X timeline
   - Monitor rate limits (100 posts/15min per user)

## Common gotchas

- **Default fields are minimal**: Posts return only `id` and `text` by default. Always add `fields` and `expansions` to get author, metrics, media, etc.
- **Expansions require matching fields**: `expansions=author_id` alone returns only the author ID. Add `user.fields=username,name` to get author details.
- **Search is limited to 7 days by default**: Use `/2/tweets/search/all` for full archive (requires pay-per-use or Enterprise).
- **Bearer Token is read-only**: Cannot post, like, or manage user data. Use OAuth for write operations.
- **Rate limits reset on a 15-minute window**: Check `x-rate-limit-reset` header; don't assume a fixed time.
- **429 errors require exponential backoff**: Start with 1-minute wait, double on each retry. Don't retry immediately.
- **Null fields are omitted**: If a field has no value, it won't appear in the response. Don't assume all requested fields are present.
- **Filtered stream rules are case-sensitive for accents**: `#cumpleaños` won't match `#cumpleanos` in stream rules (but will in search).
- **Deleted/withheld posts return 404**: Don't retry indefinitely; treat as permanent.
- **Streaming connections drop frequently**: Implement automatic reconnect with backoff; use recovery features (backfill_minutes) if available.

## Verification checklist

Before submitting work:

- [ ] Authentication credentials are correct and not hardcoded
- [ ] Bearer Token or OAuth tokens are valid and not expired
- [ ] Query syntax is tested (use recent search to validate before full archive)
- [ ] Fields and expansions are specified (not relying on defaults)
- [ ] Response includes expected data (check `data` and `includes` objects)
- [ ] Pagination is handled (save `next_token`, stop when absent)
- [ ] Rate limit headers are monitored (implement backoff for 429)
- [ ] Error responses are parsed (check `errors` array)
- [ ] Timestamps are in ISO 8601 format (e.g., `2024-01-15T10:30:00.000Z`)
- [ ] User IDs and post IDs are treated as strings (not integers)
- [ ] Sensitive data (tokens, keys) is not logged or exposed
- [ ] Streaming connections include reconnect logic
- [ ] Post text is properly escaped (special characters, newlines)

## Resources

- **[Comprehensive page listing](https://docs.x.com/llms.txt)** — Full navigation of all X API documentation
- **[X API Introduction](https://docs.x.com/x-api/introduction)** — Overview of endpoints, pricing, and features
- **[Make Your First Request](https://docs.x.com/make-your-first-request)** — Step-by-step quickstart with cURL examples
- **[Authentication Overview](https://docs.x.com/fundamentals/authentication/overview)** — Detailed guide to OAuth 1.0a, OAuth 2.0, and Bearer Token

---

> For additional documentation and navigation, see: https://docs.x.com/llms.txt