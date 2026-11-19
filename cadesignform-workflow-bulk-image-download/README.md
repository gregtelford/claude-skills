# Bulk Image Downloading Tool

## What it is

A Claude skill that reliably bulk-downloads every image in a scene, or a whole project, from [cadesignform.dk Workflow](https://workflow.cadesignform.dk/), the proprietary review site our 3rd party 3D rendering partner uses to share renders and material scans for feedback.

**Why it exists:** Workflow has limited options for bulk downloading. Individual scenes can't be downloaded in one go, so each image has to be opened and downloaded one at a time, which takes a long time when a review round contains dozens of images. This skill hands that job to Claude. Open a scene in Chrome and ask Claude to run the skill. It collects every image in the scene, downloads each one exactly once, and finishes with a file-count check so nothing is missed or duplicated. The work runs reliably in the background, so you can get on with other tasks in the meantime.

## The problem

As a brand we are often reviewing several draft outputs at a time, sometimes eclipsing 100 outputs. So downloading and reviewing images can become time consuming and has been identified as a bottleneck and an inefficient use of time. 

Bulk downloads have been considered as an option, however the website is built from an older ASP.NET WebForms app, and a few of its quirks make naive automation fail in ways that are easy to miss at first:

- Image IDs aren't assigned in scene order, so you can't just increment an ID to get "the next image."
- The Download button opens the file in a hidden background tab and gives no feedback on the page, so an automated agent assumes the click failed and clicks again, which creates duplicates.
- It's a full-page-reload app, so clicking too early lands on the *old* page's still-live Download button — producing duplicates and silent skips.
- Larger files (30MB+) fail to download when the click is scripted rather than a real, trusted click, with no error shown.
- Even doing everything right, an occasional download can still land missing roughly 1 in 20–25 times, with no signal from the page that it happened.

## The fix

This skill documents the full process:

- Pull the scene's true image ID list up front from the scene page's HTML.
- Skip the Download button entirely and request each image's download link exactly once, spaced out to allow for the site's delay.
- Always finish with a file-count check and a targeted retry for anything missing.

It grew out of several rounds of debugging real download runs (33 of 38, then 41 of 48, then 44 of 48), so the fixes in here are specific, field-tested answers to specific failure modes rather than general best practices.

## Version history

- **v2 (Nov 2026):** We found the main cause of duplicate downloads. The Download button opens the file in a hidden background tab and gives no feedback on the page, so the agent kept clicking it again, and the site's delay on large files made that worse. v2 skips the button: it requests each image once by ID through a background queue with a 12-second gap, and gets the full image list from the scene page's HTML, so lazy loading can't hide any images.
- **v1:** Navigated to each image's page, polled until the page had finished loading, then clicked the Download button and waited 10 seconds before moving on.

## Results

Running the skill allows users from our marketing team to bulk download complete scenes to their browser's Downloads folder, and do so with an agentic-first approach that allows the process to be carried out in the background while working on other tasks. Time savings are estimated to be 4-6 hours per project (all downloads, multiple rounds of revisions, generalized quantity). 

See [SKILL.md](./SKILL.md) for the full procedure.
