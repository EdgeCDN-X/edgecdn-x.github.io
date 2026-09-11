---
name: bootstrap-newsletter
description: "Prepare and generate MJML newsletter files for monthly campaigns. Automatically sources content from recent social posts or docs, generates MJML templates with branding and footer, and optionally compiles to HTML via MJML API."
argument-hint: "Month and year (e.g., 'September 2026'), or 'current month', with optional content sources"
user-invocable: true
---

# Bootstrap Newsletter

Use this skill to create professional MJML newsletter files for monthly campaigns. The skill sources content from recent posts or documentation, builds a templated MJML file, and optionally compiles to HTML.

## Outcome

- **MJML Template File** — Named `{month-short}-{year}.mjml` (e.g., `sep-2026.mjml`)
- **Organized Structure** — Sections for announcements, features, community highlights, calls-to-action
- **Branding & Footer** — EdgeCDN-X header, content sections, `listmonk-footer.html` included
- **Optional HTML Output** — Compiled HTML version (only when explicitly requested)
- **Stored** — In `docs/newsletter/{year}/` folder with version control ready
- **Preview-Ready** — MJML structure supports preview in compatible editors/clients

## Procedure

### 1. Gather Newsletter Context

Before building the newsletter:
- **Month & Year**: Which month to cover? (e.g., "September 2026" or "current month")
- **Content Source**: 
  - Auto-collect from `docs/posts/{year}/week-XX/` (default if not specified)
  - Manual content list (specific blog posts, docs pages, announcements)
  - Mix of posts + custom content
- **Newsletter Focus**: General product updates, feature spotlight, community highlight, or mixed?
- **Target Audience**: Developers, platform engineers, open-source community, customers?
- **Tone**: Formal/professional, conversational, or brand-specific?

### 2. Collect Content from Last Month

If no content is provided, auto-source from recent posts:

**Search Strategy**:
1. List all files in `docs/posts/{year}/week-{01-04}/` for the target month
2. Extract title, date, and summary from each post's frontmatter
3. Collect 4-8 key pieces for newsletter inclusion

**Content Filtering**:
- Include posts tagged with announcement, feature, update, community
- Skip meta/internal posts
- Prioritize posts with clear user value and engagement hooks

### 3. Plan Newsletter Structure

Organize MJML sections into a logical flow:

**Standard Newsletter Structure**:
1. **Header** — EdgeCDN-X branding, banner, welcome message
2. **Featured Announcement** — Top highlight (1-2 sections)
3. **Feature Spotlights** — 2-4 key updates or announcements (1 per section)
4. **Community Highlight** — User stories, contributions, or engagement summary (optional)
5. **Upcoming Events / Roadmap** — What's next (optional)
6. **Calls-to-Action** — GitHub link, docs, feedback form, etc.
7. **Footer** — listmonk-footer.html (always included)

### 4. Build MJML Template

Create the MJML structure with proper syntax:

**Template Framework**:
```mjml
<mjml>
  <mj-head>
    <mj-title>EdgeCDN-X Newsletter - [Month] [Year]</mj-title>
    <mj-preview>[Preview text - 50-100 chars]</mj-preview>
    <mj-style>
      /* Email-safe CSS for styling */
    </mj-style>
  </mj-head>
  
  <mj-body>
    <!-- Header Section -->
    <mj-section background-color="#[brand-color]">
      <mj-column>
        <mj-text font-size="28px" align="center">EdgeCDN-X Newsletter</mj-text>
        <mj-text font-size="16px" align="center">[Month] [Year]</mj-text>
      </mj-column>
    </mj-section>
    
    <!-- Content Sections from Posts -->
    [Auto-generated sections from posts]
    
    <!-- Divider -->
    <mj-section>
      <mj-column>
        <mj-divider border-color="#[accent-color]"></mj-divider>
      </mj-column>
    </mj-section>
    
    <!-- Call-to-Action -->
    <mj-section>
      <mj-column>
        <mj-button href="[link]">Read Full Article</mj-button>
      </mj-column>
    </mj-section>
    
    <!-- Footer -->
    [listmonk-footer.html content embedded]
  </mj-body>
</mjml>
```

### 5. Convert Post Content to MJML Sections

For each selected post, convert to MJML:

**Post-to-MJML Mapping**:
- **Title** → `<mj-text font-weight="bold" font-size="18px">{Post Title}</mj-text>`
- **Date** → `<mj-text font-size="12px" color="#999">{Date}</mj-text>`
- **Summary** → `<mj-text font-size="14px">{Post summary or excerpt}</mj-text>`
- **Link** → `<mj-button href="{post-url}">Read More</mj-button>`
- **Image** (if available) → `<mj-image src="{image-url}" alt="{alt-text}"></mj-image>`

**Section Template** (per post):
```mjml
<mj-section padding="20px">
  <mj-column>
    <mj-text font-weight="bold" font-size="18px">{Post Title}</mj-text>
    <mj-text font-size="12px" color="#999">{Post Date}</mj-text>
    <mj-text font-size="14px" line-height="1.6">{Post excerpt or summary}</mj-text>
    <mj-button href="{post-url}">Read Full Post</mj-button>
  </mj-column>
</mj-section>
```

### 6. Embed Footer

Include `listmonk-footer.html` at bottom:

```mjml
<mj-section>
  <mj-column>
    <!-- Read listmonk-footer.html and embed raw HTML -->
    <mj-raw>
      [Content from listmonk-footer.html]
    </mj-raw>
  </mj-column>
</mj-section>
```

**Footer Notes**:
- Typically contains unsubscribe link, company address, legal text
- Must be valid MJML/HTML (raw inclusion via `<mj-raw>`)
- Check file exists at `docs/newsletter/listmonk-footer.html`

### 7. Save MJML File

**Naming Convention**: `{month-short}-{year}.mjml`
- Examples: `sep-2026.mjml`, `oct-2026.mjml`, `dec-2026.mjml`

**Storage Path**: `docs/newsletter/{year}/`
- Full path: `docs/newsletter/2026/sep-2026.mjml`

**Commit Message**:
```
newsletter: Add September 2026 MJML template

- Auto-sourced content from week 36-38 posts
- 6 featured announcements
- Ready for HTML compilation
```

### 8. Validate MJML Syntax

Before finalizing:
- [ ] MJML structure is valid (matching opening/closing tags)
- [ ] All `<mj-*>` elements use correct attributes
- [ ] Links are properly formatted and point to current URLs
- [ ] Footer is included and properly embedded
- [ ] No broken image URLs
- [ ] Preview text is concise (50-100 characters)
- [ ] All post titles and summaries are accurate and current

### 9. Generate HTML (Optional, On Request)

**Only when explicitly asked:**

1. **Use MJML Compiler**:
   ```bash
   mjml docs/newsletter/2026/sep-2026.mjml -o docs/newsletter/2026/sep-2026.html
   ```

2. **Alternative** (if MJML CLI unavailable):
   - Use MJML API (Node.js, online renderer, etc.)
   - Generate standalone HTML file

3. **Post-Processing**:
   - Verify HTML renders correctly in test email clients
   - Check mobile responsiveness
   - Validate footer is present at bottom

4. **Store HTML**:
   - Save as `{month-short}-{year}.html` in same folder as MJML
   - Commit alongside MJML file

### 10. Provide Preview & Distribution Info

After creating MJML:
- **Preview Tip**: Open MJML file in compatible email editor for visual preview
- **Distribution**: Newsletter ready for listmonk or other email service
- **Next Steps**: Specify publish date, segment targeting, or send confirmation

## Decision Points

- **Content Source**: Auto-collect vs. manual specification
  - Auto-collect saves time; manual allows curation
  - Hybrid: Provide list of preferred posts, auto-fill remaining slots
  
- **Newsletter Length**: 4-6 posts (short) vs. 8-10 posts (comprehensive)
  - Shorter drives higher open rates; comprehensive shows activity/progress
  
- **Tone**: Match brand voice (professional, conversational, developer-friendly, etc.)

- **HTML Generation**: Only when asked (MJML is primary artifact)
  - MJML enables future updates without recompiling
  - HTML generated on-demand for distribution
  
- **Branding**: Use standard EdgeCDN-X colors, fonts, logo
  - Update branding if rebrand/refresh occurs

## Quality Criteria

- MJML syntax is valid (no malformed tags or attributes)
- All content is sourced from posts or docs (no invented content)
- Links are current and point to accessible resources
- Footer is included and properly embedded
- Preview text is compelling and under 100 characters
- Post summaries are accurate excerpts (not paraphrased incorrectly)
- Layout is visually balanced (not too dense or sparse)
- Calls-to-action are clear and actionable
- Newsletter is ready for listmonk/email service import

## Completion Summary

End with:

- **File Created**: Full path and filename (e.g., `docs/newsletter/2026/sep-2026.mjml`)
- **Content Count**: Number of posts/sections included
- **Content Sources**: Which posts/docs were included with dates
- **Naming**: Confirm naming convention used
- **Footer Status**: Confirmed listmonk-footer.html is included
- **Validation**: MJML syntax verified
- **HTML Status**: Compiled? Not compiled? (Only if asked)
- **Next Steps**: Distribution, scheduling, testing recommendations

## Related Skills

- `prepare-posts` — Create individual social media posts that feed into newsletters
- `docs-refresh` — Ensure newsletter content aligns with current documentation
- `agent-customization` — Customize newsletter branding, layout, or tone guidelines
