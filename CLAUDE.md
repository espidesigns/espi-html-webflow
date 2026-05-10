# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Purpose

Single-file (`index.html`) HTML-to-Webflow converter. Parses pasted HTML/CSS/JS and writes it to the clipboard as XSCP format (`application/json` MIME), which the Webflow Designer accepts as a paste.

## XSCP Format

`{ "type": "@webflow/XscpData", "payload": { "nodes": [], "styles": [], "assets": [], "ix1": [], "ix2": {} }, "meta": {} }`

- Node `_id` must be UUID with `_` prefix
- `classes` array holds Style `_id` UUIDs (not class name strings)
- Every node needs `data.tag`, `data.text` (bool), `data.attr.id` (even `""`)
- Text leaves: `{ _id, text: true, v: "content" }` — no `children`, no `data`
- `SKIP_TAGS` (`br`, `hr`, `wbr`) must be skipped entirely — emitting them as Block nodes inside a `text:true` container crashes Webflow
- `markTextContainers()` post-walk pass sets `data.text = true` on nodes whose direct children include text leaves
- `fixMixedContent()` post-walk pass handles two crash patterns:
  1. **Mixed content**: node has BOTH text-leaf children AND Block children (e.g. `<h1>Text <span>inline</span></h1>`). Webflow crashes because `data.text=true` nodes can't have Block siblings alongside text leaves.
  2. **Inline-only Blocks**: a text-container tag (`p`, `h1`–`h6`, `li`, etc.) has Block-only children that are all inline HTML elements (span, strong, em…). e.g. `<p><strong>Bold</strong><span>text</span></p>`. Webflow expects `<p>` to be a text container (`data.text=true`); having `data.text=false` with Block children crashes it.
  - Fix for both: collapse entire subtree into a single concatenated text leaf. Inline styling lost, text content preserved.
- **Orphaned node pruning in `assemble()`**: `fixMixedContent` replaces parent `children` arrays, leaving old Block children (strongs, spans, etc.) in the flat `nodes` array with no parent. Webflow iterates ALL nodes in the array — not just tree-reachable ones — and crashes on parentless nodes. Fix: DFS from root(s) at the end of `assemble()`, filter `nodes` to only reachable IDs before returning.

## Custom Code Embed Limit

Webflow's HtmlEmbed component has a **50,000-character limit** per embed.

Mitigation strategy (in priority order):
1. **Auto-split large CSS** — split into multiple `u-embed-css` HtmlEmbed nodes at the 49,900-char mark (leaving headroom for `<style>` wrapper tags), splitting on `}` boundaries (never mid-rule). All chunks share one `u-embed-css` Style record.
2. **Show character counts live** in the stats row — CSS and JS pill counters update on `input` events. Yellow at 45,000 chars, red at 50,000.
3. **Warn on oversized JS** — JS cannot be safely split. If JS > 50,000 chars, the red indicator signals the user to host externally (`<script src="..."></script>`).

Do NOT attempt to base64 or otherwise encode embed content — Webflow renders it verbatim.

## Clipboard Writing

`navigator.clipboard.writeText()` only writes plain text. Use `execCommand('copy')` intercepting the copy event to set `application/json` + `text/plain` MIME types. Must be called from a user gesture (button click).

## Security

Never use `innerHTML` with untrusted user content. Parse with `DOMParser`. Build the preview tree via DOM API (`createElement`, `appendChild`).

