# Task 02 — PageSpeed Insights

## How to get your PageSpeed screenshot

1. Open `orthonow-landing.html` locally in Chrome
2. Host it temporarily using one of these options:
   - **VS Code Live Server** extension → right-click → Open with Live Server
   - **Python:** `python3 -m http.server 8080` in the file's directory
   - **npx:** `npx serve .` in the file's directory
3. Go to https://pagespeed.web.dev/
4. Paste your localhost URL (or deploy to Netlify Drop for a public URL — just drag the file to https://app.netlify.com/drop)
5. Run the Mobile analysis
6. Screenshot the result and save as `pagespeed-screenshot.png` in this folder

## Why this page will score 90+

The file has zero external resource requests:
- No Google Fonts (uses system font stack)
- No external CSS frameworks
- No external JS libraries
- No images (all SVGs are inline)
- All CSS and JS are inlined in the single HTML file

LCP element: the `<h1>` tag — pure text, renders on first paint.
TBT: ~0ms — the only JS is an event listener attached after DOM load, inside an IIFE.
CLS: 0 — no images, no dynamically injected layout-shifting elements.

## Netlify Drop (fastest public URL for submission)

Drag `orthonow-landing.html` to https://app.netlify.com/drop
You get a public URL in ~10 seconds. Use that URL for PageSpeed Insights.
