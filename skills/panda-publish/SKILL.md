---
name: panda-publish
description: Publish social media content to connected platforms. Use when the user wants to post to social media, schedule a post, publish content, or share something on Twitter, Instagram, LinkedIn, Threads, Bluesky, YouTube, Pinterest, or TikTok.
license: MIT
metadata:
  author: nosypanda
---

# Social Publish

Publish social media posts to connected platforms via the Nosy Pandas API from Claude Code.

## Configuration

Configuration is stored in `~/.pandas` as simple key=value pairs:

```
api_url=https://nosypandas.com/api
api_key=your-key-here
media_folder=~/social-media
```

If this file doesn't exist or is missing values, the skill will walk the user through setup:
1. Ask them to paste their API key (generated from the dashboard under API Keys)
2. Write `~/.pandas` automatically

The user can also just paste their API key directly in chat at any time — the skill should detect it and offer to save it.

`media_folder` is optional and defaults to `~/social-media/`.

### Reading Configuration

```bash
# Read config values (use these instead of env vars in all curl commands)
PANDAS_API_URL="https://nosypandas.com/api"
PANDAS_API_KEY=$(grep '^api_key=' ~/.pandas 2>/dev/null | cut -d= -f2-)
PANDAS_MEDIA_FOLDER=$(grep '^media_folder=' ~/.pandas 2>/dev/null | cut -d= -f2-)
PANDAS_MEDIA_FOLDER="${PANDAS_MEDIA_FOLDER:-$HOME/social-media}"
```

## Available Commands

| Command | Description |
|---------|-------------|
| Publish | Create and publish a post to one or more platforms |
| History | View recent posts and their statuses |
| Detail | Check a specific post's full details |
| Retry | Retry a failed post |
| Delete | Delete a scheduled or failed post |

## Publish Flow

Follow these steps in order:

### Step 1: Verify Configuration

Read `~/.pandas` and check that `api_key` is present.

If the file doesn't exist or is missing `api_key`:
1. Ask: "Paste your API key" (tell them to generate one from the dashboard under API Keys)
2. Write the values to `~/.pandas`:
```bash
cat > ~/.pandas << 'EOF'
api_url=https://nosypandas.com/api
api_key=the-pasted-key
media_folder=~/social-media
EOF
chmod 600 ~/.pandas
```

If the user pastes what looks like an API key without being asked, detect it and offer to save it to `~/.pandas`.

### Step 2: Fetch Connected Accounts

```bash
curl -s "$PANDAS_API_URL/accounts" \
  -H "Authorization: Bearer $PANDAS_API_KEY" \
  -H "Accept: application/json"
```

**Response:**
```json
{
  "accounts": [
    { "id": 1, "platform": "twitter", "account_name": "@myhandle" },
    { "id": 2, "platform": "bluesky", "account_name": "@me.bsky.social" }
  ]
}
```

If no accounts are returned, tell the user to connect accounts via the web dashboard.

### Step 3: Select Platforms

Show the user their connected accounts as a numbered list and ask which to post to:

> Which platforms do you want to post to?
> 1. Twitter/X (@myhandle)
> 2. Bluesky (@me.bsky.social)
> 3. LinkedIn (My Profile)

After selection, show the character limits and media requirements for the selected platforms using the Platform Reference Table below.

### Step 4: Ask for Content

Ask: "What do you want to post?"

Show character limits for each selected platform so the user knows constraints before writing. If content exceeds a platform's limit and that platform supports threading (Twitter, Threads, Bluesky), note the content will be auto-split into a thread.

If content exceeds LinkedIn's 3000 character hard limit, warn the user and ask them to shorten it.

### Step 5: Platform-Specific Fields

- If **YouTube** is selected, ask: "What's the video title?" (required, max 500 chars)
- If **Pinterest** is selected, ask: "What's the destination URL for this pin?" (required, must be a valid URL)

### Step 6: Media Selection

Scan the configured media folder using `find`:

```bash
MEDIA_DIR="${PANDAS_MEDIA_FOLDER:-$HOME/social-media}"
find "$MEDIA_DIR" -maxdepth 1 -type f \( -iname "*.jpg" -o -iname "*.jpeg" -o -iname "*.png" -o -iname "*.gif" -o -iname "*.webp" -o -iname "*.mp4" -o -iname "*.mov" -o -iname "*.avi" -o -iname "*.webm" \) 2>/dev/null
```

If files are found, list them (distinguishing images vs videos by extension) and ask which to attach. If no files found, ask: "Do you want to attach any media? Provide a file path, or skip."

**Important checks before proceeding:**

- If a selected platform **requires media** (Instagram, YouTube, Pinterest, TikTok) and no media is attached, warn the user and ask them to provide media.
- If a selected platform **requires video** (YouTube, TikTok) and only images are attached, warn the user that video is required.
- Check media counts against the Platform Reference Table limits below and warn about any violations.
- If a platform has `noMix: true` and both images and videos are selected, warn the user.
- Accepted file types: jpg, jpeg, png, gif, webp (images); mp4, mov, avi, webm (videos).

### Step 7: Schedule or Publish Now

Ask: "Publish now or schedule for later?"

If scheduling, ask for the date and time. Accept natural language like "tomorrow at 9am" and convert to ISO 8601 format. Also ask for timezone if not obvious.

### Step 8: Confirmation Summary

Show a summary with any applicable warnings:

```
Content: [first 100 chars...]
Media: [count] files ([list filenames])
Platforms: Twitter, Bluesky
Timing: Publish now
Title: [if YouTube]
Link URL: [if Pinterest]

Warnings:
- Twitter: Content will be split into a thread (exceeds 280 chars)
```

Ask: "Look good? (yes/no)"

### Step 9: Execute Post

This step has two parts: stage media (if any), then create the post.

#### Step 9a: Stage Media

For each selected media file, upload it to the staging endpoint. The server handles all cloud storage interaction internally.

```bash
# Stage each file — returns a short-lived token
STAGE_RESULT=$(curl -s -X POST "$PANDAS_API_URL/media/stage" \
  -H "Authorization: Bearer $PANDAS_API_KEY" \
  -H "Accept: application/json" \
  -F "file=@/path/to/file.jpg")

TOKEN=$(echo "$STAGE_RESULT" | jq -r '.token')
```

**Response (201):**
```json
{
  "token": "stg_abc123...",
  "original_filename": "photo.jpg",
  "type": "image",
  "expires_at": "2026-03-16T14:00:00+00:00"
}
```

Tokens are valid for 2 hours. Collect all tokens for the post creation step.

#### Step 9b: Create Post

If **media was staged**, include `media_tokens` in the JSON:

```bash
curl -s -X POST "$PANDAS_API_URL/posts" \
  -H "Authorization: Bearer $PANDAS_API_KEY" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "POST_CONTENT",
    "platforms": [ACCOUNT_ID_1, ACCOUNT_ID_2],
    "publish_now": true,
    "media_tokens": ["stg_TOKEN_1", "stg_TOKEN_2"]
  }'
```

If **no media**, omit `media_tokens`:

```bash
curl -s -X POST "$PANDAS_API_URL/posts" \
  -H "Authorization: Bearer $PANDAS_API_KEY" \
  -H "Accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "POST_CONTENT",
    "platforms": [ACCOUNT_ID_1, ACCOUNT_ID_2],
    "publish_now": true
  }'
```

Optional fields (add to JSON when applicable):
- `"title": "VIDEO_TITLE"` — for YouTube
- `"link_url": "https://example.com"` — for Pinterest
- `"scheduled_at": "2026-03-16T09:00:00Z"` — for scheduled posts (omit `publish_now` or set to `false`)
- `"timezone": "America/New_York"` — timezone for scheduled posts

**Response (201):**
```json
{
  "post": {
    "id": 42,
    "content": "Hello world!",
    "status": "published",
    "platforms": [
      { "platform": "twitter", "status": "published", "url": "https://x.com/..." },
      { "platform": "bluesky", "status": "published", "url": "https://bsky.app/..." }
    ]
  }
}
```

### Step 9c: Poll for Final Status

Some platforms (especially Threads) take time to process posts. The create response may show `failed` with no error, `pending`, or `publishing` — these are non-terminal statuses that may resolve on their own. **Long threads (5+ parts) can take several minutes** because Meta processes each container sequentially.

After creating the post, check if any platform in the response has a non-terminal status:
- `pending`, `publishing`, or `failed` with `null`/missing error

If so, determine the poll count based on content:
- **Threads with threaded content** (content exceeded 500 chars and was auto-split): poll up to **12 times** with **10-second** waits (2 minutes total)
- **All other cases**: poll up to **4 times** with **5-second** waits (20 seconds total)

```bash
# Wait, then check updated status
sleep $POLL_INTERVAL
curl -s "$PANDAS_API_URL/posts/POST_ID" \
  -H "Authorization: Bearer $PANDAS_API_KEY" \
  -H "Accept: application/json"
```

**Stop polling early** if all platforms have reached a terminal status:
- `published` (success)
- `failed` with a non-null error message (genuine failure)

After polling completes (or stops early), proceed to Step 10 with whatever status we have. If a platform still shows `failed` with no error after all polls, display it as "Still processing" rather than a hard failure — the server will continue checking in the background and update the status automatically.

### Step 10: Display Results

Show the result per platform:
- Success: "Twitter/X: Published — https://x.com/..."
- Pending: "LinkedIn: Pending — check back shortly"
- Failed: "Instagram: Failed — Instagram requires media."
- Still processing: "Threads: Still processing — the server is still checking and will update the status automatically. Use the Detail command to check later."

### Step 11: Move Media

After successful posting, move used media files to a `posted/` subfolder:

```bash
MEDIA_DIR="${PANDAS_MEDIA_FOLDER:-$HOME/social-media}"
mkdir -p "$MEDIA_DIR/posted/"
mv "$MEDIA_DIR/used-file.jpg" "$MEDIA_DIR/posted/"
```

## Post History Flow

Fetch recent posts:

```bash
curl -s "$PANDAS_API_URL/posts" \
  -H "Authorization: Bearer $PANDAS_API_KEY" \
  -H "Accept: application/json"
```

Returns paginated posts with platform statuses. Display as a table showing post ID, content preview, status, and platform results.

## Post Detail Flow

Check a specific post's full details:

```bash
curl -s "$PANDAS_API_URL/posts/POST_ID" \
  -H "Authorization: Bearer $PANDAS_API_KEY" \
  -H "Accept: application/json"
```

**Response:**
```json
{
  "post": {
    "id": 42,
    "content": "Hello world!",
    "title": null,
    "status": "published",
    "scheduled_at": null,
    "created_at": "2026-03-15T12:00:00Z",
    "platforms": [
      { "platform": "twitter", "status": "published", "url": "https://x.com/...", "error": null }
    ],
    "media": [
      { "url": "https://cdn.example.com/file.jpg", "type": "image", "filename": "photo.jpg" }
    ]
  }
}
```

## Retry Flow

Retry a failed post:

```bash
curl -s -X POST "$PANDAS_API_URL/posts/POST_ID/retry" \
  -H "Authorization: Bearer $PANDAS_API_KEY" \
  -H "Accept: application/json"
```

**Response:**
```json
{
  "post": {
    "id": 42,
    "status": "pending"
  }
}
```

Only posts with `failed` status can be retried. If the post is not in a failed state, the API will return a 404.

## Delete Flow

Delete a scheduled or failed post. Only posts with `scheduled` or `failed` status can be deleted — published posts cannot be removed.

### Step 1: Identify the Post

Use the **Detail** flow to fetch the post and confirm its status is `scheduled` or `failed`. Show the post summary to the user and ask for confirmation:

> Are you sure you want to delete this post?
> Content: [first 100 chars...]
> Status: scheduled
> Platforms: Twitter, Bluesky

### Step 2: Send Delete Request

```bash
curl -s -X DELETE "$PANDAS_API_URL/posts/POST_ID" \
  -H "Authorization: Bearer $PANDAS_API_KEY" \
  -H "Accept: application/json"
```

**Response (200):**
```json
{
  "message": "Post deleted."
}
```

### Step 3: Display Confirmation

Show: "Post #POST_ID has been deleted."

If the API returns 404, the post either doesn't exist or is in a non-deletable status (e.g., published). Inform the user accordingly.

## Platform Reference Table

| Platform | Max Content | Threading | Requires Media | Requires Video | Max Images | Max Videos | No Mix |
|----------|------------|-----------|----------------|----------------|------------|------------|--------|
| Twitter/X | 280 chars | Yes | No | No | 4 | 1 | Yes |
| Bluesky | 300 chars | Yes | No | No | 4 | 1 | No |
| Threads | 500 chars | Yes | No | No | 10 | 1 | No |
| Instagram | None | No | Yes | No | 10 | 1 | No |
| LinkedIn | 3000 chars | No | No | No | 20 | 1 | No |
| YouTube | None | No | Yes | Yes | 0 | 1 | No |
| Pinterest | None | No | Yes | No | 1 | 1 | Yes |
| TikTok | None | No | Yes | Yes | 35 | 1 | Yes |

**Notes:**
- "Threading" means content exceeding the character limit is auto-split into a thread.
- "No Mix" means the platform does not allow images and videos in the same post.
- YouTube requires a `title` field (max 500 chars). Pinterest requires a `link_url` field.
- YouTube and TikTok require at least one video file — images alone will be rejected.
- Accepted file types: jpg, jpeg, png, gif, webp (images); mp4, mov, avi, webm (videos).

## Error Handling

| Status | Meaning | Recovery |
|--------|---------|----------|
| 401 | Invalid API key | Check `api_key` in `~/.pandas` is correct |
| 403 | Subscription required | Subscribe at the dashboard |
| 404 | Post not found or not retryable | Verify the post ID and that its status is `failed` |
| 422 | Validation error | Show the specific error messages from the response body and help the user fix them |
| 500 | Server error | Try again later |
