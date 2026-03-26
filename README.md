
# Articles

A collection of technical articles with platform-specific adaptations for publishing.

## Publishing Platforms

| Platform | URL | Focus |
|----------|-----|-------|
| Medium | https://medium.com/ | Narrative-driven, conversational technical content |
| Dev.to | https://dev.to/ | Developer community, Markdown-native, tutorial-style |
| SAP Community | https://community.sap.com/ | Professional, SAP-ecosystem focused |
| LinkedIn | https://www.linkedin.com/ | Thought-leadership, concise, business-relevant |

## Project Structure

```
├── README.md                  # This file
├── prompt.md                  # Prompts for adapting articles to each publishing platform
├── Cloud Native Architecture/
│   ├── README.md              # Topic overview
│   └── prompts.md             # Article generation prompts
└── SAP Commerce/
    ├── README.md              # Topic overview
    ├── prompts.md             # Article generation prompts
    ├── 1 - *.md               # Base article (platform-neutral)
    ├── medium.com/            # Medium-adapted versions
    ├── dev.to/                # Dev.to-adapted versions
    ├── community.sap.com/    # SAP Community-adapted versions
    └── linkedin.com/          # LinkedIn-adapted versions
```

## Workflow

1. **Generate a base article** using prompts from `<topic>/prompts.md`
2. **Adapt for each platform** using prompts from [`prompt.md`](prompt.md)
3. **Platform-specific versions** are saved in corresponding subdirectories (e.g., `medium.com/`, `dev.to/`)
