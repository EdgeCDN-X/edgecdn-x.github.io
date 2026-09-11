---
name: docs-refresh
description: "Refresh EdgeCDN-X docs from CDN workspace project changes and source docs. Use when updating MkDocs pages from edgecdnx-controller, edgecdnx-plugin, or s3gateway changes, validating newsletter MJML/HTML, or preparing GTM/social copy from docs changes."
argument-hint: "Changed CDN project, docs topic, feature, newsletter, or launch update"
user-invocable: true
---

# Docs Refresh

Use this skill to update EdgeCDN-X documentation from verified changes in the CDN workspace projects, then validate the docs and related distribution material.

## Outcome

- MkDocs pages in `edgecdn-x.github.io/docs/` reflect current changes from `edgecdnx-controller/`, `edgecdnx-plugin/`, and `nginx-s3-gateway/` in the CDN workspace.
- `edgecdn-x.github.io/mkdocs.yml` navigation stays aligned with added, removed, or renamed pages.
- Newsletter MJML and HTML files in `edgecdn-x.github.io/docs/newsletter/` remain paired and consistent when a docs update affects announcements.
- GTM and social copy is derived from the same verified source material as the docs.

## Procedure

1. Identify the documentation target from the user request: controller/CRDs, DNS routing, caching, S3 gateway, secure URLs, architecture, newsletter, or GTM/social copy.
2. Inspect the current project changes in the CDN workspace before editing docs:
   - `edgecdnx-controller/`: read `git status --short`, changed READMEs/docs, changed CRD/API files under `api/`, and relevant controller code when needed.
   - `edgecdnx-plugin/`: read `git status --short`, changed README/docs, CoreDNS plugin configuration, routing logic, and tests relevant to the docs topic.
   - `nginx-s3-gateway/`: read `git status --short`, changed README/docs, gateway configuration, examples, and deployment docs relevant to S3 gateway origins.
3. Read the closest existing docs page before editing. If adding a page, read `edgecdn-x.github.io/mkdocs.yml` and nearby docs in the same nav section.
4. Read the source-of-truth material for the target component:
   - Controller and CRDs: `edgecdnx-controller/README.md`, `edgecdnx-controller/api/`, `edgecdn-x.github.io/docs/crds.md`, `edgecdn-x.github.io/docs/edgecdnx-controller.md`
   - DNS routing and CoreDNS plugin: `edgecdnx-plugin/README.md`, `edgecdn-x.github.io/docs/coredns.md`, `edgecdn-x.github.io/docs/coredns-deployment.md`, `edgecdn-x.github.io/docs/edgecdnx-controller-router.md`
   - Caching layer: `edgecdn-x.github.io/docs/cache-prerequisites.md`, `edgecdn-x.github.io/docs/ingress-nginx.md`, `edgecdn-x.github.io/docs/edgecdnx-controller-cache.md`
   - S3 gateway origins: `nginx-s3-gateway/README.md`, `nginx-s3-gateway/docs/getting_started.md`, `nginx-s3-gateway/docs/development.md`, `edgecdn-x.github.io/docs/s3-gateway.md`
   - Architecture and component overview: `edgecdn-x.github.io/docs/architecture.md`, `edgecdn-x.github.io/docs/core-components.md`, `edgecdn-x.github.io/docs/index.md`
5. Compare changed source facts with the published docs. Update only the smallest set of docs needed to remove stale, missing, or contradictory information.
6. Keep existing MkDocs conventions: Markdown under `docs/`, YAML `tags:` frontmatter when useful, language-tagged fenced code blocks, and Material-compatible tabs or attributes.
7. When creating or moving docs pages, update `mkdocs.yml` in the matching nav section.
8. If the change affects newsletter or launch messaging, update or propose matching material in `docs/newsletter/` and produce channel-specific GTM/social copy from the verified docs facts.
9. Validate the result:
   - Run `mkdocs build` from `edgecdn-x.github.io/` when dependencies are available.
   - For newsletter changes, ensure each edited `.mjml` has a matching `.html`, and compile MJML to HTML when the local toolchain supports it.
   - Report skipped validation with the missing command or dependency.
   - For any edited page with tables or nested lists, check the rendered `site/<page>/index.html` for a real `<table>`/nested `<ul>`/`<ol>` (see MkDocs Markdown Rendering Rules below) rather than assuming the Markdown source is correct.

## Decision Points

- If a README and docs page disagree, prefer the implementation README or code as the current source, then note the mismatch in the summary.
- If a changed file and README disagree, prefer the changed implementation or CRD/API file, then update or flag the README/docs mismatch.
- If the user says `s3gateway`, map that to the CDN workspace project `nginx-s3-gateway/` unless they provide a different path.
- If source material is incomplete, ask for the missing product fact instead of inventing behavior.
- If a requested GTM claim is not supported by docs or code, rewrite it as a roadmap/status statement or ask for confirmation.
- If a docs update would duplicate a long explanation that already exists elsewhere, link to the existing page instead.

## MkDocs Markdown Rendering Rules

`edgecdn-x.github.io` uses plain Python-Markdown (via `mkdocs.yml` `markdown_extensions`), which is stricter than GitHub-flavored Markdown. Verified rendering pitfalls:

- **Tables require the `tables` extension.** It's enabled in `mkdocs.yml`; do not remove it. Without it, `| a | b |` rows render as literal text inside a `<p>`.
- **Tables need a blank line before them.** A table immediately following a text line (e.g. `**Label**:` then `| ... |` on the next line with no blank line in between) is treated as a paragraph continuation and never becomes a `<table>`. Always insert a blank line before the header row.
- **Nested list items need 4-space indentation, not 3.** Python-Markdown requires sub-list content indented by at least 4 spaces regardless of the parent marker width (e.g. `1. `, `2. ` is only 3 chars). A 3-space indent (`   - `) causes the nested item to flatten into the parent list as a sibling `<li>` instead of nesting inside a child `<ul>`/`<ol>`.
- When adding or editing tables or nested lists in any `docs/*.md` page, grep for `^   [-*] ` (3-space indent) and tables missing a preceding blank line before considering the edit complete.

## Quality Criteria

- Technical claims are traceable to a current CDN workspace change, README, existing docs page, CRD/API file, or implementation file.
- The docs remain task-oriented: prerequisites, configuration, examples, operational behavior, and troubleshooting appear where they help the reader complete work.
- Vocabulary is consistent: EdgeCDN-X, controller, CRDs/CRs, CoreDNS plugin, DNS based routing, ingress-nginx caching layer, S3 gateway, GitOps, locations, zones, services, DNSEndpoints, prefix lists.
- Newsletter and social copy stay specific and defensible, without unsupported production-readiness, benchmark, or enterprise claims.

## Completion Summary

End with:

- Source files read.
- CDN workspace changes inspected.
- Docs, newsletter, or GTM/social files changed.
- Validation run or skipped.
- Open questions for any unverified claims.