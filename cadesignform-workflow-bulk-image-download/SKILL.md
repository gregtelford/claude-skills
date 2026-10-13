---
name: cadesignform-workflow-bulk-image-download
description: Use when downloading all images from a scene on the cadesignform.dk Workflow review tool (e.g. Keilhauer textile/material scan review), so every image is captured without duplicates or silent misses.
---

# Bulk image download from cadesignform.dk Workflow

The Workflow tool (`workflow.cadesignform.dk`) is a classic ASP.NET WebForms app. Each scene (`Scene.aspx?id=<sceneId>`) is a lazy-loaded grid of thumbnails linking to `Comment.aspx?id=<imageId>`, which shows a Download button, a "View 1:1" link, a Timeline link, and an "Image N of M in scene" counter. This skill encodes three hard-won fixes; skipping any one of them reintroduces missed or duplicate downloads.

## Why the naive approach fails

1. **Image IDs are NOT sequential by scene position.** `id+1` is the next image uploaded, not the next image in the scene. Never increment ids or rely on "Image N of M" for ordering.
2. **This is a full-page-reload app.** After navigating to a new `Comment.aspx?id=`, the browser keeps rendering the OLD page until the new one finishes loading (the crossfade just looks like a transition). Clicking Download too early hits the old page's still-live button — causing the OLD image to download again (duplicate) while the intended one is skipped.
3. **Large files silently fail when the Download click is scripted.** The button does `window.open('/Services/DownloadImage.ashx?id=<id>', '_blank')`. A JS `element.click()` (untrusted/synthetic gesture) reliably fails to actually save files in the 30–41MB range, even though it works fine for smaller files (~11MB). A real, trusted click (via a browser automation tool's native click action) is required for large files to download reliably. Network-request logging does NOT capture these new-tab download navigations — don't rely on it to verify.
4. **Even with every fix applied, an occasional download (roughly 1 in 20–25) can still land missing** with no error signal from the page. Treat this as expected, not a sign the method is broken — the fix is a mandatory final verification + targeted retry pass, not a perfect first pass.

## Procedure

### 1. Scrape the authoritative image ID list (never trust visual scroll-counting)

On the scene grid page, run:

```js
async function collectImageIds() {
  let ids = new Set();
  for (let i = 0; i < 40; i++) {
    document.querySelectorAll('a[href*="Comment.aspx?id="]').forEach(a => {
      const m = a.getAttribute('href').match(/id=(\d+)/);
      if (m) ids.add(m[1]);
    });
    window.scrollBy(0, 800);
    await new Promise(r => setTimeout(r, 200));
  }
  document.querySelectorAll('a[href*="Comment.aspx?id="]').forEach(a => {
    const m = a.getAttribute('href').match(/id=(\d+)/);
    if (m) ids.add(m[1]);
  });
  return {count: ids.size, ids: Array.from(ids).sort((a,b)=>a-b)};
}
await collectImageIds();
```

Confirm `count` matches the page's "Filter View (N)" badge before proceeding. This sorted id list is the complete, ground-truth work queue — iterate it directly by navigating to `Comment.aspx?id=<id>` for each one. Do NOT use the in-page "›" next-arrow (click-counting drifts and silently skips/duplicates).

### 2. For each id: navigate → poll for real load → real click → wait

```js
async function waitForLoad(id, timeoutMs) {
  const start = Date.now();
  let lastHref = null;
  while (Date.now() - start < timeoutMs) {
    const tl = document.getElementById('ContentPlaceHolder1_hlImageTimeline');
    lastHref = tl ? tl.getAttribute('href') : null;
    if (lastHref && lastHref.includes('id=' + id)) return {status: 'loaded', waitedMs: Date.now() - start};
    await new Promise(r => setTimeout(r, 250));
  }
  return {status: 'timeout', waitedMs: Date.now() - start, lastHref};
}
await waitForLoad('<id>', 10000);
```

Then:
- Click the Download button using a real, trusted click (not a scripted `.click()`) at its verified screen coordinate. Scripted clicks silently fail to download large files.
- Wait a full, undisturbed ~10 seconds before navigating to the next id. Don't shorten this to speed things up — it's necessary for large files to finish writing to disk before the tab/context changes.

Note: the load-check occasionally times out (`status: 'timeout'`, `lastHref: null`) even though the page's `<title>` already shows the correct image — this just means the Timeline link populates slightly slower than the rest of the page. Proceed with the click in that case; it's still landing on the correct, fully-loaded page. This has not correlated with missed downloads in testing.

These steps can be safely batched (e.g. 3 images per batch: navigate → poll → click → wait, repeated) — batching does not reintroduce the duplicate/miss problems as long as each image still gets its own poll, real click, and full wait before the next navigation.

### 3. Mandatory final verification (do not skip, even after a clean-looking run)

After processing every id, confirm the destination folder's file count matches the scraped id count. Do not assume success from "no errors observed" — silent misses in the ~4–5% range can occur even with every fix applied correctly.

If any are missing:
- Identify which images by name (filenames like "Fern map", "Red model") — map these back to ids via the titles captured during the run (page `<title>` at each step records the material/colorway name).
- Re-download only those specific ids, one at a time (not batched), with the same navigate → poll → real click → 10s wait sequence.
- If a file is reported as duplicated rather than missing, no action is needed beyond noting it — duplicates are harmless, just delete the extra copy.
- Re-confirm the count after the retry pass. Repeat the targeted-retry step if still short.

## Quick checklist

- [ ] Scrape full id set from the scene grid; confirm count matches "Filter View (N)"
- [ ] Iterate scraped ids directly by URL — never the next-arrow, never id+1
- [ ] Poll `ContentPlaceHolder1_hlImageTimeline` href for the target id before acting (don't use a fixed wait for page-load)
- [ ] Click Download with a real click, not a scripted `.click()`
- [ ] Wait ~10s undisturbed after each click before navigating away
- [ ] After the full pass, verify final file count against the scraped id count
- [ ] Retry any missing ones individually (not batched); confirm again
