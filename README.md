# Articles

A collection of technical articles on SAP Commerce and Cloud Native Architecture.

🌐 **Published at**: [https://alextsvetkov.github.io/articles/](https://alextsvetkov.github.io/articles/)

## Project Structure

```
├── README.md                  # This file
├── _config.yml                # GitHub Pages configuration
├── index.md                   # Site landing page
├── prompt.md                  # Prompts for adapting articles to each publishing platform
├── Cloud Native Architecture/
│   ├── README.md              # Topic overview
│   └── prompts.md             # Article generation prompts
└── SAP Commerce/
    ├── README.md              # Topic overview
    ├── prompts.md             # Article generation prompts
    └── 1-15 *.md              # Articles (platform-neutral Markdown)
```

## Workflow

1. **Generate a base article** using prompts from `<topic>/prompts.md`
2. **Adapt for each platform** using prompts from [`prompt.md`](prompt.md)
3. **Publish** to the target platform (Dev.to, Medium, SAP Community, LinkedIn)