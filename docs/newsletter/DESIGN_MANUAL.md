---
title: EdgeCDN-X Newsletter Design Manual
description: Brand guidelines and MJML template standards for consistent newsletter design across all editions
version: 1.0
last-updated: 2026-09-11
---

# EdgeCDN-X Newsletter Design Manual

This document defines the visual design, branding, and MJML structure for all EdgeCDN-X newsletters. All monthly newsletters must follow this standard to maintain brand consistency.

## Brand Identity

### Color Palette

| Usage | Color | Hex Code | RGB |
| --- | --- | --- | --- |
| Primary Brand Color | EdgeCDN-X Blue | `#3472ed` | rgb(52, 114, 237) |
| Accent/Highlight | Bright Blue | `#1E90FF` | rgb(30, 144, 255) |
| Accent Links | Cyan | `#01b1ff` | rgb(1, 177, 255) |
| Background (Body) | Light Gray-Blue | `#f4f6fb` | rgb(244, 246, 251) |
| Section Background | White | `#ffffff` | rgb(255, 255, 255) |
| Text Primary | Dark Gray | `#353333` | rgb(53, 51, 51) |
| Text Secondary | Medium Gray | `#666666` | rgb(102, 102, 102) |
| Footer Background | Dark Gray | `#353333` | rgb(53, 51, 51) |

### Typography

- **Font Family**: Arial, Helvetica, sans-serif (email-safe)
- **Default Text Size**: 16px
- **Heading 2 (h2)**: 18px, `#1E90FF`, bold
- **Heading 4 (h4)**: 16px, `#1E90FF`, bold
- **Button Text**: 16px, white, bold
- **Footer Text**: 12px, white

### Logo & Branding

- **Logo URL**: `https://edgecdnx.fra1.digitaloceanspaces.com/assets/public/Logo%20Transparency%20white.png`
- **Logo Width**: 250px
- **Logo Placement**: Header section, centered, on primary brand color background
- **Logo Alt Text**: "Logo"

### Company Info

```
2026 TboTech s.r.o. - EdgeCDN-X
Slovakia
```

### Social Media Links

| Platform | URL | Icon Color |
| --- | --- | --- |
| YouTube | https://www.youtube.com/@edgecdnx | `#01b1ff` |
| X (Twitter) | https://x.com/EdgecdnX | `#01b1ff` |
| GitHub | https://github.com/EdgeCDN-X | `#01b1ff` |
| Website | https://edgecdnx.com | `#01b1ff` |

## MJML Template Structure

### Global Attributes (Head Section)

```mjml
<mj-attributes>
    <mj-all font-family="Arial, Helvetica, sans-serif" />
    <mj-text color="#353333" font-size="16px" />
    <mj-button background-color="#3472ed" color="#ffffff" />
</mj-attributes>
```

### Body Background

```mjml
<mj-body background-color="#f4f6fb">
    <!-- All sections and content here -->
</mj-body>
```

## Section Templates

### 1. Header Section (Required)

**Purpose**: Logo and brand introduction at the top of every newsletter

**Structure**:
```mjml
<!-- Header -->
<mj-section background-color="#3472ed" padding="30px">
    <mj-column>
        <mj-image
            src="https://edgecdnx.fra1.digitaloceanspaces.com/assets/public/Logo%20Transparency%20white.png"
            alt="Logo" width="250px" align="center" />
    </mj-column>
</mj-section>
```

**Specifications**:
- Background: Primary brand blue (`#3472ed`)
- Padding: 10px
- Logo: Centered, 250px wide
- Placement: Always first section

### 2. Main Content Section (Required)

**Purpose**: Feature articles, announcements, updates

**Structure**:
```mjml
<!-- Main Content -->
<mj-section background-color="#ffffff" padding="30px">
    <mj-column>
        <mj-text font-size="18px" color="#353333" line-height="1.6">
            <section style="font-family: Arial, sans-serif; line-height: 1.6; color: #333;">
                <h2 style="color: #1E90FF;">Section Title</h2>
                <h4 style="color: #1E90FF;">Subsection Heading</h4>
                <p>Content paragraph...</p>
            </section>
        </mj-text>
        <mj-image src="[image-url]" alt="[alt-text]" width="400px" align="center" href="[link]"/>
        <mj-text align="center" color="#3472ed">
            <a href="[link]" style="color:#3472ed;text-decoration:none;">
                [Link Text]
            </a>
        </mj-text>              
    </mj-column>
</mj-section>
```

**Specifications**:
- Background: White (`#ffffff`)
- Padding: 30px
- Text Color: Primary dark gray (`#353333`)
- Font Size: 16px (default), 18px (section text)
- Line Height: 1.6 (readability)
- Headings (h2, h4): Bright blue (`#1E90FF`)
- Images: 400px width, centered
- Links: Accent blue (`#3472ed`), no underline

**Content Types**:
- Article/announcement with text and image
- Multiple articles in separate sections
- Video links (with thumbnail image and clickable link)

### 3. Call-to-Action Section (Recommended)

**Purpose**: Encourage reader engagement and next steps

**Structure**:
```mjml
<!-- Call-to-Action -->
<mj-section background-color="#ffffff">
    <mj-column width="80%">
        <mj-text>
            Introductory text...
        </mj-text>
        <mj-button href="[cta-link]"
            background-color="#3472ed" color="white" border-radius="5px" padding="15px 30px">
            [Button Text]
        </mj-button>
    </mj-column>
</mj-section>
```

**Specifications**:
- Background: White (`#ffffff`)
- Button Background: Primary blue (`#3472ed`)
- Button Text: White
- Button Border Radius: 5px
- Button Padding: 15px 30px

### 4. Footer Section (Required)

**Purpose**: Social links, company info, legal

**Structure**:
```mjml
<!-- Footer -->
<mj-section background-color="#353333" padding="20px">
    <mj-column>
        <mj-social mode="horizontal" icon-size="24px" align="center">
            <mj-social-element name="youtube" href="https://www.youtube.com/@edgecdnx"
                background-color="#01b1ff" />
            <mj-social-element name="x" href="https://x.com/EdgecdnX" background-color="#01b1ff" />
            <mj-social-element name="github" href="https://github.com/EdgeCDN-X" background-color="#01b1ff" />
            <mj-social-element name="web" href="https://edgecdnx.com" background-color="#01b1ff" />
        </mj-social>

        <mj-text align="center" font-size="12px" color="#ffffff">
            2026 TboTech s.r.o. - EdgeCDN-X<br></br>Slovakia
        </mj-text>
    </mj-column>
</mj-section>
```

**Specifications**:
- Background: Dark gray (`#353333`)
- Padding: 20px
- Social Icons: 24px size, cyan background (`#01b1ff`)
- Text Color: White
- Font Size: 12px
- Alignment: Centered

## Newsletter Structure Flow

Every newsletter should follow this order:

1. **Header** — Logo on blue background
2. **Main Content Sections** (multiple) — Articles, announcements with text/images
3. **Call-to-Action** — Engagement hook and button (optional but recommended)
4. **Footer** — Social links and company info

## Design Guidelines

### Content Layout
- Maximum column width: 600px (email standard)
- Padding: 30px for main sections, 20px for footer
- Section spacing: Use separate `<mj-section>` tags (automatic margin)
- Alignment: Center-aligned for headers, left-aligned for body text

### Images
- Recommended width: 400px for in-content images
- All images must have `alt` text
- Images should be hosted on: `https://edgecdnx.fra1.digitaloceanspaces.com/assets/public/`
- Supported formats: PNG, JPG, GIF
- For clickable images: Use `href` attribute on `<mj-image>`

### Typography
- No inline styling unless necessary for email client compatibility
- Use `<h2>` and `<h4>` for section headings (colored bright blue)
- Use `<p>` for body paragraphs
- Use `<b>` for bold emphasis (not `<strong>`)
- Line-height: 1.6 for readability

### Colors in Content
- Headings: Always `#1E90FF` (bright blue)
- Body text: `#353333` (dark gray)
- Links: `#3472ed` (primary blue), no underline
- Lists: Black text on white background

### Buttons
- Background: `#3472ed` (primary blue)
- Text: White (`#ffffff`)
- Border Radius: 5px
- Padding: 15px 30px
- Font Weight: Bold
- Max 2-3 buttons per newsletter

### Responsive Design
- Single column layout (mobile-first)
- MJML handles responsive automatically
- Test in mobile email clients before sending

## File Naming & Storage

- **Format**: MJML (XML-based email template language)
- **Naming**: `{month-short}-{year}.mjml`
  - Examples: `sep-2026.mjml`, `oct-2026.mjml`, `dec-2026.mjml`
- **Location**: `docs/newsletter/{year}/`
  - Full path: `docs/newsletter/2026/sep-2026.mjml`
- **HTML Compilation**: Only on-demand (generates `.html` file in same folder)

## Frontmatter (Optional Metadata)

```yaml
---
title: "EdgeCDN-X Newsletter - September 2026"
date: 2026-09-15
author: EdgeCDN-X Team
tags:
  - newsletter
  - september-2026
  - announcements
---
```

(Frontmatter is optional if file is stored as MJML only)

## Validation Checklist

Before finalizing any newsletter:

- [ ] Header section with correct logo is present
- [ ] All content sections use correct background colors (white: `#ffffff`)
- [ ] All headings are bright blue (`#1E90FF`)
- [ ] All body text is dark gray (`#353333`)
- [ ] All images have alt text
- [ ] All images are hosted on DigitalOcean Spaces (public URL)
- [ ] All links are valid and current
- [ ] All buttons use primary blue background (`#3472ed`)
- [ ] Footer section with social links is present
- [ ] Footer company info is correct and centered
- [ ] MJML syntax is valid (no malformed tags)
- [ ] File named correctly: `{month-short}-{year}.mjml`
- [ ] File stored in correct location: `docs/newsletter/{year}/`
- [ ] Line breaks and spacing are consistent
- [ ] No broken image links
- [ ] No inline CSS except where necessary for email compatibility

## Email Client Compatibility

This design is optimized for:
- Gmail (web & mobile)
- Outlook (web & desktop)
- Apple Mail
- Thunderbird
- Mobile clients (iOS Mail, Gmail Mobile, Outlook Mobile)

**Best Practices**:
- Arial/Helvetica font ensures compatibility
- MJML framework handles CSS fixes automatically
- Avoid advanced CSS features (animations, gradients)
- Test in multiple email clients before sending

## Related Files

- **Bootstrap Newsletter Skill**: `.github/skills/bootstrap-newsletter/SKILL.md` — Workflow for creating newsletters using this design
- **Prepare Posts Skill**: `.github/skills/prepare-posts/SKILL.md` — Create individual post content for newsletter sections
- **April 2026 Newsletter**: `docs/newsletter/apr-2026.mjml` — Reference example (source of this design)

## Contact & Questions

For design feedback or updates to this manual, contact the EdgeCDN-X documentation team.

---

**Document Version**: 1.0  
**Last Updated**: 2026-09-11  
**Next Review**: 2026-12-11
