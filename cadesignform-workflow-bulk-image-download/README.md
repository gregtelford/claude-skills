# cadesignform-workflow-bulk-image-download

A Claude skill that reliably bulk-downloads every image in a scene from [cadesignform.dk Workflow](https://workflow.cadesignform.dk/), the review tool we use at Keilhauer for material and textile scan review.

## The problem

Workflow is an older ASP.NET WebForms app, and a few of its quirks make naive automation fail in ways that are easy to miss at first:

- Image IDs aren't assigned in scene order, so you can't just increment an ID to get "the next image."
- It's a full-page-reload app, so clicking too early lands on the *old* page's still-live Download button — producing duplicates and silent skips.
- Larger files (30MB+) fail to download when the click is scripted rather than a real, trusted click, with no error shown.
- Even doing everything right, an occasional download can still land missing roughly 1 in 20–25 times, with no signal from the page that it happened.

## The fix

This skill documents the full process: scraping the scene's true image ID list up front, polling for a confirmed page load before acting (instead of a fixed wait), using a real click for the Download button, and — critically — always finishing with a file-count verification and a targeted retry pass for anything missing.

It grew out of several rounds of debugging real download runs (33 of 38, then 41 of 48, then 44 of 48, before landing on a fully reliable process), so the fixes in here are specific, field-tested answers to specific failure modes rather than general best practices.

See [SKILL.md](./SKILL.md) for the full procedure.
