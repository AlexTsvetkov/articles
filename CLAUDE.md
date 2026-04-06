# CLAUDE.md — Project Instructions for Claude

> This file provides context and instructions for Claude (Claude Code, Cline, or any AI agent) when working in this repository.

## Project Overview

This is a **technical articles repository** for dev.to blog posts. Articles cover:
- **SAP Commerce Cloud** — architecture, ImpEx, FlexibleSearch, performance, deployment, extensions, catalogs, OCC API, composable storefront, data migration, caching, testing, troubleshooting, upgrades, type system
- **Cloud Native Architecture** — SAP BTP, CAP, Kyma, Kubernetes, microservices

## Repository Structure

```
articles/
├── CLAUDE.md                  # This file — AI agent instructions
├── README.md                  # Public project readme
├── index.md                   # Article index/table of contents
├── prompt.md                  # Prompts used to generate articles
├── promotion.md               # (local-only, gitignored) Promotion strategy playbook
├── .gitignore
├── SAP Commerce/
│   ├── README.md
│   ├── prompts.md
│   └── 1-*.md through 15-*.md  # 15 SAP Commerce articles
└── Cloud Native Architecture/
    ├── README.md
    └── prompts.md
```

## dev.to Integration

- **Username:** `aliaksandr_tsviatkou`
- **Profile:** https://dev.to/aliaksandr_tsviatkou
- **Dashboard:** https://dev.to/dashboard
- **API key:** See `promotion.md` (local-only file)

### dev.to API Quick Reference

```bash
# List all published articles
curl -H "api-key: <KEY>" https://dev.to/api/articles/me/published?per_page=100

# Get single article
curl -H "api-key: <KEY>" https://dev.to/api/articles/<ID>

# Update article
curl -X PUT -H "api-key: <KEY>" -H "Content-Type: application/json" \
  -d '{"article": {"title": "...", "tags": ["tag1","tag2"]}}' \
  https://dev.to/api/articles/<ID>

# Create article
curl -X POST -H "api-key: <KEY>" -H "Content-Type: application/json" \
  -d '{"article": {"title": "...", "body_markdown": "...", "published": false}}' \
  https://dev.to/api/articles
```

## Conventions

### Article Files
- Filename format: `<number>-<Title_With_Underscores>.md`
- Articles are numbered in reading order within each series
- Content is in standard Markdown compatible with dev.to
- Each article folder has its own `README.md` (series overview) and `prompts.md` (generation prompts)

### Writing Style
- Technical and practical — include code examples, diagrams, and real-world scenarios
- Target audience: intermediate to senior developers working with SAP technologies
- Each article should be self-contained but reference related articles in the series
- Use dev.to-compatible Markdown (liquid tags for embeds, `{% %}` syntax)

### Git Workflow
- `main` branch for published content
- Local-only files (`promotion.md`) are in `.gitignore`
- Commit messages: `Add article: <title>` or `Update article: <title>`

## Common Tasks

### Publishing a New Article
1. Write the article as a `.md` file in the appropriate folder
2. Update `index.md` with the new article entry
3. Update the folder's `README.md` with the article in the series listing
4. Publish to dev.to via API (set `published: true`)
5. Commit the `.md` file to git

### Promoting Articles
- See `promotion.md` for the complete AI-agent-executable promotion playbook
- Promotion tasks include: tag optimization, title optimization, cover images, series organization, engagement hooks, cross-posting, community engagement, analytics review

### Updating an Existing Article
1. Edit the `.md` file locally
2. Push the update to dev.to via API using the article ID
3. Commit changes to git

## Important Notes

- **Never commit API keys or secrets** — they belong only in `promotion.md` (gitignored)
- **Always ask for user approval** before changing published article titles or body content on dev.to
- **Tags and series updates** are low-risk and can be done without explicit approval
- **Rate limiting:** dev.to API limits requests. Add 1-second delays between API calls. On HTTP 429, wait 30 seconds and retry.