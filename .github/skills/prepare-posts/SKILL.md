---
name: prepare-posts
description: "Create engaging social media posts for product announcements, features, or community engagement. Generates multi-post campaigns with platform-specific content, competitive positioning, and audience engagement strategies."
argument-hint: "Topic, week/timeframe, key features, or campaign theme"
user-invocable: true
---

# Prepare Social Media Posts

Use this skill to create high-quality social media posts for product launches, feature announcements, or community engagement campaigns. The skill produces multiple posts with different angles, messaging strategies, and engagement hooks.

## Outcome

- 2-4 standalone social media posts (as Markdown files with frontmatter)
- Each post has a distinct angle and messaging strategy
- Posts include competitive positioning and/or alternative comparisons
- Posts include engagement hooks (questions, calls-to-action)
- Posts stored in organized folder structure: `docs/posts/YEAR/week-XX/`
- Metadata (title, date, tags) included for blogging/scheduling systems
- Optional: Emoji-free or platform-specific formatting applied

## Procedure

### 1. Gather Campaign Context
Before drafting posts, clarify:
- **Topic/Announcement**: What product feature, update, or message are we promoting?
- **Timeframe**: Week number, dates, or publication schedule (e.g., "week 38, Sept 15-18")
- **Key Differentiators**: What makes this feature/product stand out?
- **Target Audience**: Developers, platform engineers, CTOs, open-source community?
- **Competitive Context**: What are users currently using? (SaaS competitors, open-source alternatives, DIY solutions)

### 2. Define Post Strategy
Decide on positioning across 2-4 posts:

**Post A — Feature Announcement** (Overview/Leadership angle)
- Lead with benefit and differentiation
- Highlight 3-4 key capabilities
- Mention competitive alternatives for context
- Include 1 engagement hook (direct question about current solution)

**Post B — Technical Deep-Dive** (Problem-Solution angle)
- Position as solution to common pain points
- Explain the "how" behind key features
- Contrast vendor lock-in vs. open-source/self-hosted benefits
- Include multiple-choice or scenario-based engagement questions

**Post C — Use Case / Success Story** (Optional; if relevant)
- Real-world application or customer scenario
- Show tangible outcomes or benefits
- Engagement: Ask about similar challenges in audience's environment

**Post D — Community/Roadmap** (Optional; long-term planning)
- Share vision and roadmap milestones
- Invite contribution and feedback
- Engagement: Ask what problems users want solved next

### 3. Draft Post Content

For each post, follow this structure:

```markdown
---
title: "Clear, Engaging Title - 8-12 Words"
date: YYYY-MM-DD
tags:
  - primary-topic
  - feature-area
  - platform
  - engagement-angle
---

[Opening Hook]
[Problem/Context]
[Solution/Features - 3-5 bullet points]
[Competitive Positioning or Differentiation]
[Engagement Question or Call-to-Action]
[Link to Relevant Docs]
```

**Writing Guidelines:**
- **Headline**: Benefit-first, specific, searchable
- **Opening**: Grab attention; use relatable language or bold statement
- **Body**: 3-5 key points (concise, bulleted for scannability)
- **Engagement**: Direct question, multiple-choice options, or scenario-based prompt
- **CTA**: Link to docs, sign-up, comment thread, or specific next step
- **Length**: 250-500 words per post (platform-friendly)
- **Formatting**: Bold for emphasis, lists for readability

### 4. Apply Metadata and Styling

- **Frontmatter**:
  - `title`: Descriptive post title (8-12 words, searchable keywords)
  - `date`: Publication date (YYYY-MM-DD format)
  - `tags`: 3-5 relevant tags for categorization and search
  
- **Formatting**:
  - Remove emojis unless platform-specific brand guidelines require them
  - Use bold (`**text**`) for key points and section headers
  - Use dashes (`-`) or numbers for lists
  - Hyperlink relevant docs or resources inline

### 5. Organize and Store

- Create folder: `docs/posts/YYYY/week-XX/` (week number or date range)
- Name files: `post-1.md`, `post-2.md`, etc. (or thematic names if preferred)
- Commit to repository with meaningful message (e.g., "posts: Add week 38 social media content")

### 6. Validate Posts

Before finalizing:
- [ ] Each post has a clear, distinct angle or messaging strategy
- [ ] Posts complement each other without redundancy
- [ ] Engagement hooks are specific (avoid generic "what do you think?")
- [ ] Competitive positioning is fair and defensible
- [ ] Links point to relevant, current documentation
- [ ] Metadata is consistent and searchable
- [ ] No broken markdown formatting
- [ ] Post length is appropriate for target platforms (250-500 words)

## Decision Points

- **Single vs. Multiple Angles**: Use 1-2 posts for quick announcements; 3-4 posts for major campaigns spread over a week
- **Competitive Positioning**: Reference specific competitors (Route 53, Cloudflare, etc.) only if comparison is defensible and adds value
- **Engagement Style**: Direct questions work for Twitter/LinkedIn; multiple-choice or scenarios work better on technical forums
- **Emoji Usage**: Follow brand guidelines; avoid unless intentionally playful. Remove if brand is more professional
- **Link Targets**: Always link to current, relevant documentation or product pages (never broken links)

## Quality Criteria

- Each post has a unique angle and messaging strategy (not just rephrasing)
- Engagement hooks are specific enough to drive meaningful responses
- Competitive positioning is accurate and defensible (backed by real product differences)
- Posts are written in a consistent voice aligned with brand tone
- Metadata (date, tags) enable easy discovery and scheduling
- Posts avoid marketing jargon; use clear, accessible language
- CTAs are specific and actionable (e.g., "Read our CoreDNS guide" vs. "Learn more")

## Completion Summary

End with:

- **Posts Created**: List file names and angles
- **Folder Structure**: Where posts are stored
- **Metadata**: Dates, tags, and scheduling info
- **Validation**: Confirmation all quality criteria met
- **Next Steps**: Suggestions for platform-specific adaptation (LinkedIn, Twitter, Dev.to, etc.) if needed

## Related Skills

- `docs-refresh` — Ensure posts reflect current documentation
- `agent-customization` — Customize brand voice or engagement strategy
- `project-setup-info-local` — Set up blog/posts publishing infrastructure
