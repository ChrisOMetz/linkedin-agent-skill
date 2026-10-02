---
name: li-publish
description: >-
  Send an approved LinkedIn post to FeedHive, the only route this user allows
  for posting to LinkedIn. Creates a FeedHive draft or a scheduled post through
  the FeedHive MCP server (or the official CLI as fallback), then confirms it
  landed. Use when the user says "draft"/"schedule" after a li-post receipt,
  "send it to FeedHive", "queue this", "publish this", "post this to
  LinkedIn", or asks what is scheduled, drafted or published in FeedHive. Also
  use for pulling FeedHive analytics for li-audit.
---

# li-publish

FeedHive is the user's social media automation tool, and **every post to their
LinkedIn account goes through it and nowhere else.** No browser automation, no
direct LinkedIn API calls, no other scheduler, no "I'll just paste it for you".
If FeedHive cannot do something, say so and stop. That rule exists because
LinkedIn restricts accounts driven by unofficial automation, and FeedHive posts
through LinkedIn's sanctioned API.

## When this skill may write

Only after the user has seen the exact final text (post-`/li-human`) and
answered with one of:

- **draft** - create it in FeedHive as a draft. Nothing goes live. This is the
  default if their reply is only "yes" or "ok".
- **schedule** (with or without a time) - create it as `scheduled` at the
  agreed time. This is the only path that publishes without another click.

Never infer "schedule" from a general "looks good". Publishing to a public
profile is hard to take back, so the user says the word. Editing or deleting
anything already `scheduled` or `published` needs a fresh confirmation too.
FeedHive has **no publish-now operation**: "post it now" means schedule a few
minutes ahead, and the user still has to say schedule.

## Pick the route

**1. FeedHive MCP server (preferred).** If the session has tools named
`mcp__<server>__feedhive_*` (the server is usually called `FeedHive`, endpoint
`https://mcp.feedhive.com`), use them. The API key lives in the user's MCP
config, so nothing else is needed, and the server has guards built in:

| tool | note |
| --- | --- |
| `feedhive_socials_list` | find the LinkedIn account id |
| `feedhive_posts_create` | draft by default; `status: "scheduled"` additionally needs `confirm_scheduling: true` |
| `feedhive_posts_get` / `feedhive_posts_list` | verify, and check for duplicates |
| `feedhive_posts_update` | replace semantics; always needs `confirm_update: true` |
| `feedhive_posts_delete` | needs `confirm: true` |
| `feedhive_media_upload_start` / `_complete` | the file `PUT` is not done by the tool, see Media |
| `feedhive_slots_*`, `feedhive_plan_assign_next` | recurring plan slots; assigning needs `confirm_scheduling: true` |
| `feedhive_analytics_post` / `_social` | numbers for `/li-audit` |

Set a `confirm_*` flag only because the user said the word in this
conversation, never to get past the prompt.

The same server also exposes **triggers** as `trigger_<id>` tools. These are
Workflows the user built in FeedHive; the description says what each does
(for example "Creates a draft post" taking `text` and `media_urls`). Use one
only when the user names it, read the description and schema first, and do not
assume a trigger is draft-only unless its description says so.

If the MCP server is configured (`claude mcp list`) but its tools are not in
the session, it was added mid-session. Tell the user to restart the session or
reconnect via `/mcp`. Do not conclude it is unavailable.

**2. Official CLI (fallback).** Pinned to the version this skill was reviewed
against (it only talks to `https://api.feedhive.com`, Node 20+):

```bash
npx -y @feedhive/cli@0.1.7 <resource> <action> [args] [options]
# posts list|get|create|update|delete, socials list, media create-upload |
# complete-upload, analytics post|social, plan-slots ... , --body-file <json>
```

The key is read from `FEEDHIVE_API_KEY` or `~/.feedhive/agent-tools.env` (one
line: `FEEDHIVE_API_KEY=fh_...`). Never ask the user to paste the key into chat,
never print it, never put it on a command line. On "Missing API key", tell the
user to add it to that file themselves (FeedHive: Settings > Account > API
Key, or Settings > Workspace if LinkedIn is connected to a workspace) and
stop. The CLI has no confirm flags, so you supply the discipline: nothing
scheduled, updated or deleted without the user's word.

Anything else (browser, curl against LinkedIn, another tool) is off the table.

## Workflow

1. **Find the target account:** list socials, keep `platform: "linkedin"` with
   `status: "active"`. If there are several (profile and company page), ask
   once and record the id in `~/.claude/linkedin/feedhive.md`. If the status is
   `failed`, report `error_message`; the fix is a reconnect in the FeedHive
   app, not a workaround.
2. **Build the payload.** Text exactly as approved, blank lines preserved, no
   @-mentions of people. Fields: `text`, `accounts: [<id>]`, `status`
   (`draft` | `scheduled`), `notes` (e.g. the hook used). For a schedule add
   `scheduled_at`: ISO 8601 **UTC**, in the future, converted from the user's
   timezone; show both in the receipt. No time given? Use the one from the
   `/li-post` receipt or `/li-plan`, else ask. With the CLI, write the body to a
   temp file with the Write tool (shell `echo` mangles newlines and quotes),
   outside the user's project.
3. **Check for a duplicate:** list drafts and scheduled posts first. Creates are
   not idempotent, so after any error or timeout list again before retrying.
4. **Create it.**
5. **Verify:** get the post and report `status`, `scheduled_at` and
   `approval.status`. `pending` means the workspace needs approval and API
   callers can never approve, so the user must approve it in the FeedHive app;
   it will not publish until they do.
6. **Log it** in `~/.claude/linkedin/log.md`: date, FeedHive post id, status,
   hook used, first line. `/li-audit` joins on that id.

```
FEEDHIVE  draft created
id:        <post id>
account:   <display name> (LinkedIn)
status:    draft | scheduled for Tue 8:15 Lisbon (07:15Z) | pending approval
```

## What FeedHive cannot do for LinkedIn

Say so plainly when it comes up; do not route around it.

- **No PDF / carousel (document) posts.** LinkedIn's API does not support them,
  so `/li-carousel` output cannot go through FeedHive.
- **No @-mentions of people**, only companies. Strip them from the text.
- **No LinkedIn groups.**
- **No comments, replies or DMs.** The API covers posts, labels, media, plan
  slots, social accounts and analytics, so `/li-comment`, `/li-reply`, `/li-dm`
  and `/li-inbox` stay copy-and-paste by the user.
- **First comment with a link** is unverified. `subposts` are thread replies
  and the docs do not say whether they become LinkedIn comments. Do not rely on
  it; ask the user first.

## Media

Allowed: `image/png`, `image/jpeg`, `image/gif`, `video/mp4`,
`video/quicktime`. Three steps:

1. Start an upload (`feedhive_media_upload_start`, or CLI `media create-upload`)
   with `filename` and `content_type`. It returns an `upload_id` and a signed
   `upload_url` valid for 10 minutes. Do not show the URL to anyone.
2. `PUT` the raw file bytes to `upload_url` with the same `Content-Type`
   (Bash `curl --upload-file`). The MCP tool does not do this step.
3. Complete the upload (`feedhive_media_upload_complete` / `media
   complete-upload`). It returns the media `id`; put that in `media: [...]`.
   The `upl_...` upload id is not a media id and the API rejects it.

## Editing, deleting, analytics

- Update has **replace semantics**: every field you send overwrites the old
  value, so send the whole intended state.
- Delete only on explicit request, after naming which post.
- Analytics for `/li-audit`: one item per publication, `metrics` varies by
  platform, `stale: true` is a cached fallback. Only posts published through
  FeedHive have FeedHive analytics.
- Rate limit is 60 requests per minute; list before you loop.
