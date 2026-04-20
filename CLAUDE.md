# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Single-file X (Twitter) post manager (`x-post-manager.html`) — a self-contained HTML/CSS/JS app with a Japanese UI, designed to help the author plan, AI-generate, and schedule promotional posts for the product "haisonikko" (a delivery-log app for light-freight drivers). There is no build step, no package manager, no tests, and no server component.

## Running / Developing

- Open `x-post-manager.html` directly in a browser (double-click or `file://`). No dev server is required.
- To test AI calls, open the ⚙️ 設定 tab and paste an API key for Groq, Gemini, or Anthropic. Keys live only in `localStorage`.
- To iterate, edit the file and reload the browser tab. Use DevTools console for debugging; all state is on `window` (e.g. `state`, `activeTagFilter`).
- There are no lints, no tests, and no CI. Do not add tooling unless the user asks — the app's single-file portability is a feature.

## Architecture

The entire app is inside `x-post-manager.html` and follows a simple render-on-event pattern — no framework, no reactivity, no modules.

**State model** (persisted to `localStorage` as `xpm_v2`):
```
state = {
  settings: { name, target, features, url, hashtags, extra },
  posts:    [ { id, theme, text, posted, scheduledDate, tags[], createdAt } ]
}
```
Keys `xpm_apikey` and `xpm_provider` are stored separately so they aren't included in export/import JSON.

**Tabs and their render entry points** — `switchTab(tab)` shows/hides `#tab-<name>` divs and calls the matching renderer:
- `calendar` → `renderCalendar()` builds a month grid, then `selectDay()` renders the day detail.
- `posts` → `renderPosts()` applies `activeTagFilter` then rebuilds the list.
- `generate` → `refreshRewriteSelect()` repopulates the rewrite dropdown.
- `settings` → `loadSettingsForm()` reads state + localStorage into the form fields.

**Mutation flow**: every user action mutates `state` in place, calls `save()` (writes JSON to localStorage), then re-calls the relevant `render*` function. There is no diffing — the innerHTML of each section is rebuilt wholesale. Preserve this pattern when adding features.

**AI provider abstraction** lives in a single function: `callClaude(msg)` (despite the name, it routes to Groq / Gemini / Anthropic based on `xpm_provider`). All three providers are called **directly from the browser**:
- Groq: `llama-3.3-70b-versatile` via OpenAI-compatible `/v1/chat/completions`.
- Gemini: `gemini-2.0-flash` via `generativelanguage.googleapis.com`.
- Anthropic: `claude-sonnet-4-6` with the `anthropic-dangerous-direct-browser-access: true` header — this is intentional for a local tool with a user-supplied key.

When adding or upgrading models, update `callClaude`, the provider `<select>` in the settings tab, and `providerHints()` (placeholder + signup link + free-tier note) together.

**Prompt construction**: `buildSystemPrompt()` interpolates `state.settings` into a fixed Japanese prompt template enforcing 140–200 chars, 口語体, URL + hashtags at the end. All three providers receive the same system prompt; Gemini gets it prepended to the user message since that API has no system role here.

**Tag system**: tags are free-form strings parsed from space/comma/、/，-separated input via `parseTags()`. Badge colors are deterministic from a string hash (`tagColors()`), so the same tag always renders the same color. `activeTagFilter` is AND-matched case-insensitively in `renderPosts()`. Clicking a badge anywhere calls `filterByTag()`, which appends to the filter and switches to the posts tab.

**XSS safety**: user content flows into `innerHTML` in several renderers. Always route untrusted strings through `esc()` (the small HTML-entity escaper near the bottom of the script) — never interpolate raw `p.text`, `p.theme`, or tag strings into template literals.

## Conventions

- UI strings and comments are **Japanese**. Match the existing tone when adding user-visible text; keep code identifiers in English.
- Keep everything in `x-post-manager.html`. No external JS/CSS files, no bundler, no npm. External resources are limited to Google Fonts via `@import`.
- Back-compat for existing users' localStorage matters: on load, missing fields are filled in (see the `state.posts.forEach(p=>{ if(!p.tags) p.tags=[]; })` migration at the bottom of the script). Add similar guards when introducing new fields rather than bumping the storage key.
- The product defaults seeded on first run (haisonikko name / features / URL / hashtags) are intentional — the app ships pre-configured for its author. Don't blank them out.
- Character counts use `[...text].length` (code-point count) to match how X counts Japanese text, not `.length`.
