# Prompts for Adapting Articles to Publishing Platforms

Use these prompts to adapt any generated article (or article prompt) to the specific formatting, tone, and audience expectations of each publishing platform.

---

## Medium (https://medium.com/)

### Adapt an Article for Medium
```
Take the following article and adapt it for publishing on Medium.com. Apply these adjustments:

- **Title**: Make it compelling and curiosity-driven. Medium titles work best when they promise value (e.g., "How I…", "The Complete Guide to…", "X Things I Learned About…").
- **Subtitle**: Add a subtitle that provides additional context and hooks the reader.
- **Opening**: Start with a personal hook, a story, or a provocative statement. Medium readers respond to narrative-driven intros rather than dry overviews.
- **Formatting**:
  - Use short paragraphs (2-3 sentences max).
  - Use pull quotes or bold key phrases to break up long sections.
  - Use headers (H2/H3) every 3-5 paragraphs for scannability.
  - Use backtick code blocks for all code/configuration examples.
  - Add horizontal rules (---) between major sections for visual separation.
  - Use numbered lists and bullet points generously.
- **Images**: Suggest where to add images or diagrams (with placeholder descriptions). Medium articles with images get significantly more engagement.
- **Tone**: Conversational and experience-driven. Write as if sharing knowledge with a peer over coffee. Use "I" and "you" naturally.
- **Call to Action**: End with a question or invitation to discuss in the comments. Optionally suggest readers follow you or clap if they found it useful.
- **Tags**: Suggest 5 relevant Medium tags for discoverability (e.g., Programming, Software Development, SAP, Cloud, Architecture).
- **Length**: Medium's sweet spot is 7-10 minute reads (~1,800-2,500 words). If the original is longer, suggest how to split into a series.

Article to adapt:
[PASTE ARTICLE OR ARTICLE PROMPT HERE]
```

### Adapt an Article Prompt for Medium
```
Take the following article prompt and modify it so the generated article will be optimized for Medium.com publishing. Add these requirements to the prompt:

- The article should open with a personal anecdote or narrative hook
- Use a conversational, first-person tone throughout
- Keep paragraphs short (2-3 sentences)
- Include suggested Medium tags at the end
- Target 7-10 minute read length (~1,800-2,500 words)
- End with an engaging question for readers
- Suggest a compelling title and subtitle pair
- Note where images or diagrams should be placed

Original prompt:
[PASTE ARTICLE PROMPT HERE]
```

---

## Dev.to (https://dev.to/)

### Adapt an Article for Dev.to
```
Take the following article and adapt it for publishing on Dev.to. Apply these adjustments:

- **Title**: Keep it direct and developer-friendly. Dev.to titles work best when they are clear and searchable (e.g., "Understanding X: A Practical Guide", "How to Build X with Y").
- **Front Matter**: Add Dev.to front matter metadata:
  ---
  title: "Article Title"
  published: true
  description: "A brief description (max 100 chars)"
  tags: tag1, tag2, tag3, tag4
  cover_image: https://placeholder-url.com/image.png
  ---
- **Formatting**:
  - Use Markdown throughout (Dev.to natively supports Markdown).
  - Use fenced code blocks with language identifiers (```java, ```xml, ```json, etc.).
  - Use {% raw %} Liquid tags where appropriate: `{% tip %}`, `{% warning %}`, `{% collapsible %}` for enhanced content blocks.
  - Use headers (## and ###) to structure content clearly.
  - Embed GitHub gists for longer code examples if applicable.
- **Tone**: Technical but friendly. Dev.to is a developer community — write as if explaining to a fellow developer. Casual is fine but substance matters most.
- **Series**: If the article is long (>3000 words), suggest splitting into a series and add series front matter: `series: "Series Name"`.
- **Engagement**: 
  - Add a brief "TL;DR" or summary section at the top for scanners.
  - End with "What's your experience with [topic]?" or similar community-engaging question.
  - Mention relevant Dev.to community tags/organizations if applicable.
- **Tags**: Suggest 4 Dev.to tags (max 4 allowed). Use popular tags like: `javascript`, `webdev`, `java`, `tutorial`, `beginners`, `architecture`, `cloud`, `sap`, `devops`, `testing`.
- **Cross-references**: Suggest linking to related Dev.to articles or documentation.
- **Length**: Dev.to works well with all lengths. For long articles, ensure good heading structure for the auto-generated table of contents.

Article to adapt:
[PASTE ARTICLE OR ARTICLE PROMPT HERE]
```

### Adapt an Article Prompt for Dev.to
```
Take the following article prompt and modify it so the generated article will be optimized for Dev.to publishing. Add these requirements to the prompt:

- Output in Markdown format with Dev.to front matter (title, published, description, tags, cover_image)
- Include a TL;DR section at the top
- Use fenced code blocks with language identifiers for all code examples
- Use a technical but approachable tone
- Suggest 4 relevant Dev.to tags
- If the article would exceed 3000 words, suggest a series split
- End with a community engagement question
- Structure with clear headers for Dev.to's auto table of contents

Original prompt:
[PASTE ARTICLE PROMPT HERE]
```

---

## SAP Community (https://community.sap.com/)

### Adapt an Article for SAP Community
```
Take the following article and adapt it for publishing on SAP Community (community.sap.com). Apply these adjustments:

- **Title**: Use a clear, professional title. SAP Community titles should be descriptive and include relevant SAP product names for searchability (e.g., "SAP Commerce Cloud: How to Optimize FlexibleSearch Queries").
- **Blog Type**: Indicate whether this should be a "Technical Article" or "Blog Post" based on content depth.
- **Opening**: Start with a clear problem statement or context. SAP Community readers are typically looking for solutions — state what the article addresses upfront.
- **Formatting**:
  - Use SAP Community's rich text editor formatting (headers, bold, numbered lists).
  - Use the code block feature for all code/configuration examples.
  - Use tables where comparing options or configurations.
  - Use screenshots or diagrams of SAP tools (HAC, Backoffice, Cloud Portal) where relevant.
  - Use callout boxes for important notes, warnings, or tips.
- **SAP-Specific Context**:
  - Reference official SAP documentation (help.sap.com) and SAP Notes where applicable.
  - Use correct SAP product names and terminology (e.g., "SAP Commerce Cloud" not "Hybris", "Composable Storefront" not just "Spartacus").
  - Mention relevant SAP versions/releases.
  - Tag relevant SAP product areas.
- **Tone**: Professional and authoritative. SAP Community values expertise and thoroughness. Write as a subject matter expert sharing knowledge with the ecosystem.
- **Tags/Topics**: Suggest relevant SAP Community topic tags (e.g., "SAP Commerce Cloud", "SAP BTP", "Cloud Native", "Integration", "SAP S/4HANA").
- **Engagement**: 
  - End with clear next steps or further reading recommendations.
  - Invite readers to share their experiences or ask questions.
  - Link to related SAP Community content if applicable.
- **Compliance**: Ensure content follows SAP Community guidelines — no promotional content, focus on technical value.

Article to adapt:
[PASTE ARTICLE OR ARTICLE PROMPT HERE]
```

### Adapt an Article Prompt for SAP Community
```
Take the following article prompt and modify it so the generated article will be optimized for SAP Community publishing. Add these requirements to the prompt:

- Use professional, authoritative tone appropriate for SAP ecosystem
- Use correct SAP product naming conventions throughout
- Reference official SAP documentation (help.sap.com) where applicable
- Include SAP version/release context
- Start with a clear problem statement
- Suggest relevant SAP Community topic tags
- Include references to SAP Notes or KBAs where relevant
- Use tables for comparing configurations or options
- End with next steps and further reading from SAP sources
- Indicate whether content fits "Technical Article" or "Blog Post" format

Original prompt:
[PASTE ARTICLE PROMPT HERE]
```

---

## LinkedIn (https://www.linkedin.com/)

### Adapt an Article for LinkedIn
```
Take the following article and adapt it for publishing as a LinkedIn Article. Apply these adjustments:

- **Title**: Make it professional and value-driven. LinkedIn titles should signal expertise and relevance to career growth (e.g., "What Every Developer Should Know About X", "Lessons Learned from X in Production").
- **Opening**: Start with a strong hook in the first 2-3 lines — this is what appears in the feed preview. Use a bold statement, a surprising statistic, or a relatable professional challenge.
- **Formatting**:
  - Keep paragraphs very short (1-3 sentences). LinkedIn readers often skim on mobile.
  - Use bold text for key takeaways and important phrases.
  - Use bullet points and numbered lists for easy scanning.
  - Use headers to break the article into clear sections.
  - For code examples, keep them minimal and well-explained — LinkedIn's audience is broader than pure developer platforms. Use screenshots of code rather than code blocks if the formatting is critical.
  - Use emojis sparingly (🔑, 💡, ⚡, 🚀) to highlight key points if it fits your personal brand.
- **Tone**: Professional thought-leadership. Write from a position of experience and authority. Share insights, opinions, and lessons learned. LinkedIn rewards personal perspective and career-relevant advice.
- **Audience Consideration**: LinkedIn's audience includes developers, architects, managers, and business stakeholders. Balance technical depth with accessibility — explain "why it matters" not just "how it works."
- **Key Takeaways**: Include a "Key Takeaways" or "TL;DR" section with 3-5 bullet points summarizing the main points.
- **Personal Angle**: Weave in personal experience, lessons from your career, or team experiences. LinkedIn rewards authenticity and storytelling.
- **Call to Action**: End with:
  - A thought-provoking question to drive comments
  - An invitation to connect or follow for more content
  - Suggest reposting if readers found it valuable
- **Hashtags**: Add 3-5 relevant LinkedIn hashtags (e.g., #SAPCommerce #CloudNative #SoftwareArchitecture #DevOps #TechLeadership).
- **Length**: LinkedIn articles perform best at 1,000-2,000 words. If the original is longer, suggest how to extract the most impactful sections or create a series.

Article to adapt:
[PASTE ARTICLE OR ARTICLE PROMPT HERE]
```

### Adapt an Article Prompt for LinkedIn
```
Take the following article prompt and modify it so the generated article will be optimized for LinkedIn Article publishing. Add these requirements to the prompt:

- Use professional thought-leadership tone
- Open with a compelling hook that works in LinkedIn's feed preview (first 2-3 lines)
- Keep paragraphs short (1-3 sentences) for mobile readability
- Balance technical depth with business relevance — explain "why it matters"
- Include personal experience or lessons-learned angle
- Add a "Key Takeaways" section with 3-5 bullet points
- Minimize code examples; focus on concepts, patterns, and decision-making
- Target 1,000-2,000 words
- End with a discussion question and 3-5 LinkedIn hashtags
- If the topic is very technical, suggest splitting into a technical article (for Dev.to/Medium) and an insights article (for LinkedIn)

Original prompt:
[PASTE ARTICLE PROMPT HERE]
```

---

## General: Adapt to All Platforms at Once

### Generate Platform-Specific Versions from One Article
```
I have the following article. Please create adaptation notes for publishing on each of these 4 platforms. For each platform, provide:
1. A suggested title optimized for that platform
2. A brief description of what structural/tone changes are needed
3. What to add (e.g., front matter, tags, hashtags, images)
4. What to remove or condense
5. Suggested tags/hashtags for discoverability

Platforms:
- Medium.com
- Dev.to
- SAP Community (community.sap.com)
- LinkedIn

Article:
[PASTE ARTICLE HERE]
```

### Adapt Any Article Prompt for a Specific Platform
```
Take the following article prompt and create 4 modified versions, one optimized for each publishing platform:
1. Medium.com — conversational, narrative-driven, 7-10 min read
2. Dev.to — Markdown-native, developer-focused, with front matter and TL;DR
3. SAP Community — professional, SAP-terminology-correct, with SAP documentation references
4. LinkedIn — thought-leadership, concise, business-relevant with personal angle

For each version, adjust the tone, length, formatting requirements, and engagement elements to match the platform's audience and best practices.

Original prompt:
[PASTE ARTICLE PROMPT HERE]