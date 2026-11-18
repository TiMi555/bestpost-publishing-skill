---
name: bestpost-publishing
description: Use BestPost MCP to query monitored accounts and posts, publish authorized content, and verify publication results.
---

# BestPost MCP

Use this Skill when the user asks to query monitored BestPost account/post data, publish content, or check a publication through the connected BestPost MCP server. The MCP server connection and token must already be configured in the AI client. Never ask the user to paste a token into chat.

## Monitoring queries

- For account profiles or performance questions, call `list_monitor_accounts`. Filter by platform (`facebook`, `instagram`, `youtube`, or `tiktok`) or account name when helpful. Results are paginated; follow the returned `page`, `pageSize`, and `total` metadata.
- For post or content performance questions, call `list_monitor_posts`. Use the account ID/name, platform, title, or date filters to narrow the query. If no date range is specified, the tool uses the latest 30 days. Date ranges cannot exceed 366 days.
- For a specific result, call `get_monitor_post` with the returned BestPost internal `postId`. It returns caption, duration, language, platform post ID, account identity, metric definitions and HTTPS references only for recognized social-platform domains. Anonymous accessibility is not verified. Use the exact platform post ID or ask the user to confirm a mapping before matching it to a local video/version; never match by title alone.
- When comment evidence matters, call `list_monitor_post_comments`. It only returns comment text already stored by BestPost. Check its `availability`: an empty list can mean no rows were collected, a permission/sync gap, or a zero platform counter; completeness is not recorded. Do not claim there are no comments unless the result supports that conclusion.
- When a trend/history question matters, call `list_monitor_post_snapshots`. These existing daily rows contain cumulative counters observed that day; they are not guaranteed 24-hour/72-hour/7-day post-age windows. Missing dates are not backfilled.
- In metric objects, `value: 0` is a stored zero and `value: null` is unknown. Stored zeros may be upstream defaults; nulls do not identify whether a metric is unsupported, permission-limited, or not synced. Keep these states distinct in summaries.
- Keep publication time separate from metric update time. `publishedAt` is rendered in `Asia/Shanghai`; source update timestamps have no stored timezone. Account `interactionRate` is an aggregate of up to the 10 latest collected videos, not a fixed 30-day or per-post rate, and its formula varies by platform/sync path.
- Treat publication-task associations as exact only when project, account, normalized platform, and remote post ID match. A missing association is `null`; never guess one from a similar title.
- The MCP token fixes the project scope. Do not ask for or send `projectId`; never try to query another project.
- These tools return read-only monitoring data and omit platform credentials and private contact details.

## Required workflow

1. Call `list_publish_accounts` before preparing a request. Match accounts by both platform and account identity; use only the returned `accountIds`. Do not substitute account IDs from `list_monitor_accounts`. If the intended account is unclear, ask which one to use.
2. Confirm the content, target accounts, and any platform-specific options that are missing. Do not silently substitute an account or platform.
3. Call `preview_publish` with the exact intended payload. Explain any validation errors or warnings and resolve them before submitting.
4. Submit only when the user explicitly asked you to publish the content. Call `submit_publish` with the previewed payload and a stable `requestKey` containing 8–100 letters, digits, underscores, or hyphens.
5. Treat a successful submission response as queued, not published. Save its returned job IDs and call `get_publish_status` to verify the final result. Report the status returned by BestPost; never claim success while a job is queued, processing, or unverified.
6. If a submit call times out or its result is unclear, query status before retrying. Reuse the same `requestKey` for a retry of the same logical request to avoid duplicate publications.

## Payload guide

Always include `platform` and `accountIds` from `list_publish_accounts`.

- Facebook and Instagram: include `mediaType` (`TEXT`, `IMAGE`, `CAROUSEL`, `VIDEO`, or `REELS`). Add `caption` and `mediaUrls` as required by the chosen format. Every media URL must be public HTTPS and accessible to BestPost's workers.
- YouTube: include `title` and `videoUrl`. Add `description`, `tags`, `privacyStatus` (`private`, `unlisted`, or `public`), and `madeForKids` when appropriate. The video URL must be public HTTPS and accessible to BestPost's workers.
- TikTok: include `videoUrl` and `postMode` (`DIRECT_POST` to publish directly, or `MEDIA_UPLOAD` to send the video to TikTok drafts). Add `caption` when needed. `allowComment`, `allowDuet`, and `allowStitch` default to `true`; set them only to match the user's requested interaction settings. Set `isBrandOrganic`, `isBrandedContent`, and `isAiGenerated` when applicable. Optional cover settings are `customThumbnailUrl` (public HTTPS) or `thumbnailOffsetMs` (0–600000).
- `preview_publish` does not create a job. `submit_publish` creates jobs in the asynchronous publishing queue.

## Uploading local media

When the client can access a local media file and make an HTTP multipart request, upload it to BestPost before previewing the publication:

1. Derive the upload URL by replacing the trailing `/mcp` in the configured MCP server URL with `/mcp/media`.
2. Send `POST` with `Authorization: Bearer <the configured project MCP token>` and multipart field `file` containing the local file. Do not send a `projectId`; the server derives the project from the token.
3. Use `data.url` from the response as `mediaUrls` or `videoUrl`, then follow the normal preview, explicit authorization, submission, and status-check workflow.

The endpoint accepts JPG, PNG, WebP, GIF, MP4, MOV, and WebM files up to 50 MB. It returns a public HTTPS URL and never returns storage credentials. Do not expose the configured MCP token in prompts, commands shown to the user, or logs. If the client cannot make multipart HTTP requests, use the existing BestPost web uploader instead.

## Safety and privacy

- The token is a long-lived project publishing credential. Never include it in prompts, post content, logs, or responses.
- The connected token is bound to one project. Do not attempt to override the project or use IDs not returned by the account-list tool.
- Ask before publishing whenever the user's request does not clearly authorize publication or leaves the content/account ambiguous.
- Report platform errors from the task status accurately. A queue acceptance or a running worker is not proof that the social platform published the post.
