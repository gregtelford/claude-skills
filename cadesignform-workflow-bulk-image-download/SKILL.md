---
name: cadesignform-workflow-bulk-image-download
description: Use when downloading all images from a scene (or a whole project) on the cadesignform.dk Workflow review tool, so every image is captured once, without duplicates or silent misses.
---

# Bulk image download from cadesignform.dk Workflow

The Workflow tool (`workflow.cadesignform.dk`) is a classic ASP.NET WebForms app used for review rounds on renders and material scans. A project (`Project.aspx?id=<projectId>`) contains scenes (`Scene.aspx?id=<sceneId>`). Each scene is a grid of thumbnails, and each thumbnail links to an image viewer page, `Comment.aspx?id=<imageId>`.

This skill runs through Claude in Chrome, inside the user's signed-in browser session.

**Default: download every image, approved or not.** Unapproved images come out watermarked, which is expected. Filter to approved only when the user asks.

## How the site works, and why the obvious approach fails

1. **The viewer's Download button gives no visible feedback.** It calls `window.open('/Services/DownloadImage.ashx?id=<imageId>', '_blank')`, which opens a background tab the agent can't see, and the viewer page doesn't change. An agent that clicks it sees nothing happen, assumes the click failed, and clicks again. Every extra click is another copy of the same file. This was the main source of duplicates.
2. **It's a full-page-reload app.** If you click "next" and then Download before the new page finishes loading, you hit the old page's still-live button. The previous image downloads again and the intended one gets skipped.
3. **Image IDs aren't sequential by scene position.** `id+1` is the next image uploaded, not the next image in the scene. Never increment IDs and never count "next" arrow clicks.
4. **Files are large and slow.** Full-size images run about 30–46 MB, and the server can take several seconds or more to start sending one.
5. **Results with the Download button have been inconsistent.** A native click worked in earlier runs but saved nothing in a later test. Requesting the download URL directly, once per image, worked reliably. The procedure below never touches the button.
6. **Scene pages have no ZIP option.** The "Full-size images (ZIP)" button is only on the project page, and it bundles every scene in the project.
7. **Silent misses can still happen.** No method has been perfect on the first pass, so a final count check and a targeted retry are mandatory.

## Procedure: one scene

### 1. Get the authoritative image list

Open the scene page (`Scene.aspx?id=<sceneId>`) in its own tab and leave it there. Leaving the page stops the download script.

Fetch the scene's HTML instead of reading the live grid. The live grid lazy-loads as you scroll, but the fetched HTML lists every image. Each image is linked more than once, so dedupe:

```js
const sceneId = new URLSearchParams(location.search).get('id');
const html = await fetch('/Scene.aspx?id=' + sceneId).then(r => r.text());
const doc = new DOMParser().parseFromString(html, 'text/html');
const seen = new Set(), items = [];
doc.querySelectorAll('a[href*="Comment.aspx"]').forEach(a => {
  const id = (a.getAttribute('href').match(/id=(\d+)/) || [])[1];
  const name = a.textContent.trim().replace(/\s+/g, ' ').replace(/^.*?\d+ x \d+ px\s*/, '');
  if (!id) return;
  if (!seen.has(id)) { seen.add(id); items.push({ id, name }); }
  else { const it = items.find(x => x.id === id); if (it && !it.name && name) it.name = name; }
});
window.__cadItems = items;
({ scene: doc.title, count: items.length, names: items.map(i => i.name) })
```

Confirm `count` matches the "N images" figure the site shows for that scene before going on.

Tip: Claude in Chrome's tool output blocks text containing URLs with query strings. Have scripts return IDs, counts and names, never full URLs.

### 2. Request each image exactly once

Each image is requested once through a hidden iframe pointed at the download endpoint, 12 seconds apart. The loop runs in the background, so the tool call returns right away:

```js
window.__cad = { total: window.__cadItems.length, done: [], state: 'running' };
(async () => {
  for (const it of window.__cadItems) {
    if (window.__cad.stop) break;
    const f = document.createElement('iframe');
    f.style.display = 'none';
    f.src = '/Services/DownloadImage.ashx?id=' + it.id;
    document.body.appendChild(f);
    window.__cad.done.push(it.id);
    await new Promise(r => setTimeout(r, 12000));
  }
  window.__cad.state = 'finished';
})();
'started ' + window.__cadItems.length
```

Never run this twice for the same scene. That would duplicate every file.

If Chrome asks whether the site may "download multiple files", the user needs to click **Allow**. They only have to do this once.

### 3. Monitor

Poll `({state: window.__cad.state, done: window.__cad.done.length, total: window.__cad.total})` every 30 seconds or so. Expect about 12 seconds per image, so a 38-image scene takes about 8 minutes. After `state` reads `finished`, wait about 30 more seconds for the last file to arrive.

### 4. Verify, then do a targeted retry (don't skip this)

Files land in Chrome's Downloads folder, which the agent can't see. Ask the user to compare the file count against `count` from step 1.

- **Missing files:** match them by name against `__cadItems`, then request only those IDs with the same iframe method. Never re-run the whole scene.
- **Duplicates:** these are harmless. The user can delete the extra copies.

Confirm the count again after the retry.

## Procedure: whole project

On the project page, use **"Full-size images (ZIP)"** in the Download Images panel. Uncheck "Include only approved images" to get everything, unless the user asked for approved only. Click it once and don't click it again, because a large project's ZIP can take a while to start.

## Approved images only

This hasn't been field-tested for scenes yet. Approved tiles show a green check marker. Filter `__cadItems` to approved tiles, then confirm the count with the user before starting.

## Guardrails

- This is a client review tool. Never click Approve, Reject, comment, status, or any other review control.
- Never click an image's Download button, and never request the same image ID twice in one pass.
- Avoid anything that raises a browser alert or confirm dialog. If one appears, stop and tell the user.
- If a step fails twice, stop and report instead of looping.

## Quick checklist

- [ ] Open the scene in its own tab and leave it there
- [ ] Fetch the scene HTML, dedupe the IDs, and check the count against the site's image count
- [ ] Start the iframe queue once (12-second gaps) and never re-run it
- [ ] Poll until it's finished, then wait about 30 more seconds
- [ ] Have the user verify the file count
- [ ] Retry only the missing IDs, then re-verify
