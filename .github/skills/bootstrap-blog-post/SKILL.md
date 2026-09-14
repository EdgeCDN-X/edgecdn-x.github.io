---
name: bootstrap-blog-post
description: "Create a concise, fact-checked Markdown blog article for the Strapi-based EdgeCDN-X blog. Use when asked to bootstrap, draft, or prepare a quick blog post from a topic, feature, release, repository change, or documentation page."
argument-hint: "Topic or source material, with optional title, audience, author, category, and publish date"
user-invocable: true
---

# Bootstrap Blog Post

Create one focused Markdown article that is ready for review and later import into the Strapi article collection. Keep the post useful and easy to finish: enough context to reward the click, without turning a quick update into a long-form guide.

## Outcome

- One fact-checked Markdown article of roughly 350-650 words
- A specific title, an excerpt no longer than 80 characters, and a URL-safe slug
- Optional author, category, and cover references when they are known
- A short opening, scannable body, and one clear next step
- Storage at `docs/blog/YYYY/DD-title-slug.md`
- A draft ready for editorial review; do not publish it to Strapi unless explicitly asked

## Procedure

### 1. Establish the Brief

Extract these details from the request:

- Topic or announcement
- Reader and the problem they care about
- Main takeaway
- Relevant source files or URLs
- Publication date, defaulting to the current date
- Author, category, and cover, when supplied

Do not block on optional metadata. If the topic, reader, or main takeaway cannot be inferred, ask one concise question covering only the missing essentials.

### 2. Verify the Content

Use the closest authoritative sources in this order:

1. User-provided facts and links
2. Current project documentation
3. Implementation, tests, release notes, and configuration
4. Repository history when timing or intent matters

Confirm product names, capabilities, commands, links, and comparisons. Never invent customer outcomes, benchmarks, release status, quotes, or roadmap commitments. Omit unsupported claims or mark them clearly for review.

### 3. Choose the Angle

Build the post around one reader benefit. Prefer this sequence:

1. Open with the practical problem or outcome in 2-3 sentences.
2. Explain what changed or what the product provides.
3. Support it with 2-4 concrete details.
4. Close with one relevant documentation, repository, trial, or contact action.

For a release, emphasize what changed and who benefits. For a feature, explain the problem and how the feature works. For a technical concept, use one concrete scenario and link to deeper documentation rather than reproducing it.

### 4. Prepare Strapi Metadata

Use YAML frontmatter matching the article model:

```markdown
---
title: "Specific, benefit-led title"
description: "Plain-language summary no longer than 80 characters."
slug: specific-benefit-led-title
date: YYYY-MM-DD
author: author-name
category: category-name
cover: relative/or-published-image-path
---
```

Rules:

- Keep `description` at 80 characters or fewer.
- Generate `slug` from the title using lowercase ASCII words and hyphens.
- Use a two-digit day in the filename.
- Include `author`, `category`, and `cover` only when verified or explicitly supplied.
- Treat author and category values as editorial references; Strapi resolves them as relations during import.
- Use meaningful alt text whenever the body contains an image.

### 5. Draft the Article

Write approximately 350-650 words unless the user requests another length. Use:

- One direct opening paragraph
- Two or three descriptive `##` headings
- Short paragraphs, usually 2-4 sentences
- A short list only when it improves scanning
- One concrete example, workflow, or technical detail
- One closing call to action linked to the most relevant current resource

Use plain, confident language for a technical audience. Expand an acronym on first use. Avoid filler, generic hype, repeated conclusions, fake urgency, and unsupported competitor claims. Do not add social-media copy or scheduling sections unless requested.

### 6. Save the Draft

Create the file at:

```text
docs/blog/YYYY/DD-title-slug.md
```

For example, a post dated September 14, 2026 titled "Health-Aware Edge Routing" becomes:

```text
docs/blog/2026/14-health-aware-edge-routing.md
```

Create the year directory when needed. If the target filename already exists, do not overwrite it; choose a more specific slug or ask before replacing the draft.

### 7. Validate

Before finishing, check:

- The post has one clear reader benefit and fulfills the supplied brief.
- Every factual statement is supported by a current source.
- The title is specific and the description is at most 80 characters.
- The slug and filename are lowercase, ASCII, and hyphen-separated.
- The year and two-digit day match the publication date.
- Optional relation and media metadata is either verified or omitted.
- Links use current published URLs and resolve when link checking is available.
- Markdown structure is valid and contains no placeholders.
- The body is concise, scannable, and roughly 350-650 words.
- The final paragraph gives the reader one useful next step.

## Decision Points

- **Missing content details:** Ask only when the topic or intended takeaway is unclear; infer tone and structure from nearby published content.
- **Insufficient evidence:** Narrow the claim or leave it out instead of filling gaps with plausible copy.
- **Long source material:** Select one angle and link to the full documentation.
- **Unknown Strapi relations:** Omit author and category rather than guessing relation values.
- **Cover unavailable:** Omit it; do not use a decorative placeholder image.
- **Publishing requested:** Confirm the target Strapi environment and credentials workflow before creating or updating an article.

## Completion Summary

Report the created file, title, word count, sources used, and any omitted or unresolved Strapi metadata. State that the post remains a draft unless publication was explicitly completed.

## Related Skills

- `prepare-posts` - Adapt a published article into platform-specific social copy
- `bootstrap-newsletter` - Include published articles in a monthly newsletter
- `docs-refresh` - Verify product details and documentation links before drafting