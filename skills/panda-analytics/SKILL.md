---
name: panda-analytics
description: View social media analytics and insights. Use when the user wants to check post performance, engagement metrics, best posting times, follower stats, content decay, posting frequency, or any analytics data from their connected platforms.
license: MIT
metadata:
  author: nosypanda
---

# Social Analytics

View social media analytics and insights via the Nosy Pandas API from Claude Code.

## Configuration

This skill shares the same `~/.pandas` config file as `panda-publish`. See that skill for the full setup flow.

Required values: `api_url` and `api_key`.

### Reading Configuration

```bash
# Read config values (use these instead of env vars in all curl commands)
PANDAS_API_URL=$(grep '^api_url=' ~/.pandas 2>/dev/null | cut -d= -f2-)
PANDAS_API_KEY=$(grep '^api_key=' ~/.pandas 2>/dev/null | cut -d= -f2-)
```

If `~/.pandas` doesn't exist or is missing values:
1. Ask: "What's your Nosy Pandas URL?" (e.g. `https://nosypandas.com`)
2. Ask: "Paste your API key" (tell them to generate one from the dashboard under API Keys)
3. Write the values to `~/.pandas`:
```bash
cat > ~/.pandas << 'EOF'
api_url=https://nosypandas.com/api
api_key=the-pasted-key
media_folder=~/social-media
EOF
chmod 600 ~/.pandas
```

If the user pastes what looks like an API key without being asked, detect it and offer to save it to `~/.pandas`.

## Available Commands

| Command | Description |
|---------|-------------|
| Overview | Post analytics with engagement metrics (paginated) |
| Best Time | Optimal times to post based on historical performance |
| Content Decay | How engagement drops off over time |
| Daily Metrics | Day-by-day engagement stats (filterable by date range) |
| Follower Stats | Follower growth and demographics |
| Post Timeline | Engagement timeline for posts |
| Posting Frequency | How often you're posting and consistency |

## Pro Plan Gate

All 7 analytics endpoints require a Pro plan. Before displaying any analytics data, you must handle the 403 response.

When any analytics API call returns a 403 status with `{"message": "Pro plan required."}`, do the following:

1. Do NOT show the raw error or JSON response
2. Display this message exactly:

> **Analytics requires a Pro plan.**
>
> You're currently on the Basic plan. Analytics — including post performance, best posting times, follower growth, and content decay — are available on Pro.
>
> Upgrade to Pro ($79/year) from your dashboard's Billing page to unlock analytics.

3. STOP. Do not retry, do not offer workarounds, do not show empty states or dummy data.

## Overview

Post analytics with engagement metrics.

### Steps
1. Verify `~/.pandas` config has `api_url` and `api_key`
2. Make the API call
3. If 403, show the Pro Plan Gate upgrade message and stop
4. Format and display results

### API Call

```bash
curl -s "$PANDAS_API_URL/analytics" \
  -H "Authorization: Bearer $PANDAS_API_KEY" \
  -H "Accept: application/json"
```

Optional query parameters:
- `?postId=ID` — filter to a specific post
- `?page=N` — pagination (default page 1)

For paginated results, offer to fetch the next page if more data is available.

## Best Time

Optimal times to post based on historical performance.

### Steps
1. Verify `~/.pandas` config has `api_url` and `api_key`
2. Make the API call
3. If 403, show the Pro Plan Gate upgrade message and stop
4. Format and display results

### API Call

```bash
curl -s "$PANDAS_API_URL/analytics/best-time" \
  -H "Authorization: Bearer $PANDAS_API_KEY" \
  -H "Accept: application/json"
```

## Content Decay

How engagement drops off over time.

### Steps
1. Verify `~/.pandas` config has `api_url` and `api_key`
2. Make the API call
3. If 403, show the Pro Plan Gate upgrade message and stop
4. Format and display results

### API Call

```bash
curl -s "$PANDAS_API_URL/analytics/content-decay" \
  -H "Authorization: Bearer $PANDAS_API_KEY" \
  -H "Accept: application/json"
```

## Daily Metrics

Day-by-day engagement stats.

### Steps
1. Verify `~/.pandas` config has `api_url` and `api_key`
2. Make the API call
3. If 403, show the Pro Plan Gate upgrade message and stop
4. Format and display results

### API Call

```bash
curl -s "$PANDAS_API_URL/analytics/daily-metrics" \
  -H "Authorization: Bearer $PANDAS_API_KEY" \
  -H "Accept: application/json"
```

Optional query parameters:
- `?days=30` — number of days to look back (default varies by API)

## Follower Stats

Follower growth and demographics.

### Steps
1. Verify `~/.pandas` config has `api_url` and `api_key`
2. Make the API call
3. If 403, show the Pro Plan Gate upgrade message and stop
4. Format and display results

### API Call

```bash
curl -s "$PANDAS_API_URL/analytics/follower-stats" \
  -H "Authorization: Bearer $PANDAS_API_KEY" \
  -H "Accept: application/json"
```

## Post Timeline

Engagement timeline for posts.

### Steps
1. Verify `~/.pandas` config has `api_url` and `api_key`
2. Make the API call
3. If 403, show the Pro Plan Gate upgrade message and stop
4. Format and display results

### API Call

```bash
curl -s "$PANDAS_API_URL/analytics/post-timeline" \
  -H "Authorization: Bearer $PANDAS_API_KEY" \
  -H "Accept: application/json"
```

Optional query parameters:
- `?postId=ID` — filter to a specific post's timeline

## Posting Frequency

How often you're posting and consistency.

### Steps
1. Verify `~/.pandas` config has `api_url` and `api_key`
2. Make the API call
3. If 403, show the Pro Plan Gate upgrade message and stop
4. Format and display results

### API Call

```bash
curl -s "$PANDAS_API_URL/analytics/posting-frequency" \
  -H "Authorization: Bearer $PANDAS_API_KEY" \
  -H "Accept: application/json"
```

## Display Guidelines

When presenting analytics data to the user:

- **Tables** for tabular data (daily metrics, post lists, overview results)
- **Bullet points** for insights (best time recommendations, frequency analysis)
- **Highlight key numbers** — top-performing posts, growth trends, peak engagement times
- **Summaries first, details on request** — lead with the headline stat, offer to drill down
- **Paginated results** (overview) — offer to fetch the next page if more data is available
- **Date ranges** — when showing daily metrics, clearly label the date range covered

## Error Handling

| Status | Meaning | Recovery |
|--------|---------|----------|
| 401 | Invalid API key | Check `api_key` in `~/.pandas` is correct |
| 403 | Pro plan required | Show the upgrade message (see Pro Plan Gate section) |
| 422 | Validation error | Show the specific error messages from the response body and help the user fix them |
| 500 | Server error | Try again later |
