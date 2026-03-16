# Nosy Pandas Skills

Claude Code skills for the [Nosy Pandas](https://nosypandas.com) social media scheduling and publishing platform.

## Skills

| Skill | Description | Version |
|-------|-------------|---------|
| panda-publish | Publish, schedule, retry, and delete social media posts | 0.1.0 |
| panda-analytics | View post performance, engagement metrics, and posting insights | 0.1.0 |

## Installation

```bash
npx @anthropic-ai/claude-code skills add /path/to/Nosy-Pandas-Skills/skills/panda-publish
npx @anthropic-ai/claude-code skills add /path/to/Nosy-Pandas-Skills/skills/panda-analytics
```

## Configuration

Both skills read from a shared `~/.pandas` config file. On first use the skill will walk you through setup, or you can create it manually:

```
api_url=https://nosypandas.com/api
api_key=your-key-here
media_folder=~/social-media
```

- **api_url** — The Nosy Pandas API base URL (default: `https://nosypandas.com/api`)
- **api_key** — Your API key, generated from the [dashboard](https://nosypandas.com) under **API Keys**
- **media_folder** — Local folder scanned for images/videos to attach to posts (default: `~/social-media`)

## Supported Platforms

- Twitter/X
- Bluesky
- Threads
- Instagram
- LinkedIn
- YouTube
- Pinterest
- TikTok

## Skill Overviews

### panda-publish

Publish social media content to connected platforms directly from Claude Code.

- **Publish** — Create and publish a post to one or more platforms (with optional media, scheduling, and threading)
- **History** — View recent posts and their statuses
- **Detail** — Check a specific post's full details
- **Retry** — Retry a failed post
- **Delete** — Delete a scheduled or failed post

### panda-analytics

View social media analytics and insights (requires Pro plan).

- **Overview** — Post analytics with engagement metrics
- **Best Time** — Optimal times to post based on historical performance
- **Content Decay** — How engagement drops off over time
- **Daily Metrics** — Day-by-day engagement stats
- **Follower Stats** — Follower growth and demographics
- **Post Timeline** — Engagement timeline for posts
- **Posting Frequency** — How often you're posting and consistency

## Versioning

Each skill declares its version in the `version` field of its `SKILL.md` frontmatter, following [semver](https://semver.org).

## License

MIT
