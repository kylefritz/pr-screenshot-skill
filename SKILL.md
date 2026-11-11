---
name: pr-screenshots
description: Put screenshots in a GitHub PR description, hosted in the repo itself (hidden-ref image hosting, no bucket), so it works for private repos. Use whenever you open or update a PR that changes anything visible (UI, layout, copy, emails, a rendered page, a chart, CLI output), when someone asks for screenshots on a PR or to add them to old PRs, when a picture would explain a change faster than words, or when you need an image in a PR, issue or comment.
---

# PR screenshots

Images beat words. A reviewer who opens a PR on a phone, or reads it over SSH, or just doesn't
want to run the branch, should see what changed without running anything. So:

- **Every PR that changes something you can see gets screenshots**, in its **description** under
  `## Screenshots`, never in a comment (comments scroll away; the description is what people read
  and what gets linked).
- **Reach for a picture for non-UI changes too, whenever it explains faster:** a before/after of a
  rendered page or email, a terminal render of a CLI's new output, a chart of a timing or size
  change (test suite 212 s → 59 s, bundle 1.4 MB → 380 KB), a diagram of a new data flow.

Upload with `scripts/pr-screenshots` (next to this file; spell out its absolute path in each
command). It needs `git`, `gh` (logged in, with push access) and Python 3. Run it from inside a
clone of the repo.

```sh
pr-screenshots before.png after-phone.png            # PR for the current branch; appends to ## Screenshots
pr-screenshots --caption "Menu, phone, empty cart" menu.png
pr-screenshots --replace shots/*.png                 # redo the whole section
pr-screenshots --pr 12 --print timing-chart.png      # upload and print markdown only (for a comment or issue)
pr-screenshots --dry-run shots/*.png
```

Markdown goes to stdout, progress to stderr. Images (png, jpg, gif, webp, svg) embed; video and PDF
come out as plain links.

## How it works: hidden-ref image hosting

The images go into the repo's own object store, but onto no branch. The script writes each file
with git plumbing (`hash-object -w`, `mktree`, `commit-tree`) and pushes that commit to
`refs/pr-screenshots/<pr>`. A ref outside `refs/heads` and `refs/tags`:

- doesn't show in the branch list or the PR's diff, and never lands in a normal clone;
- fires no Actions (push events only fire for branches and tags);
- is served by GitHub like any commit, so `https://github.com/<owner>/<repo>/raw/<sha>/<file>`
  works. For a private repo that URL redirects a logged-in viewer with access to a short-lived
  tokened URL; `raw.githubusercontent.com/...` links 404 there, so never use them.

URLs are pinned to the commit SHA. A re-run for the same PR makes a new commit whose parent is the
previous one and whose tree carries the previous files forward, so links already in the body keep
resolving. If a remote or a sandbox refuses pushes outside `refs/heads` (some cloud sandboxes and
proxies do), the script falls back to a `screenshots/pr-<pr>` branch: one extra line in the branch
list, nothing else.

The description block is marked with `<!-- pr-screenshots:begin -->` / `end`, so a re-run edits
only its own block. A first run replaces a hand-written `## Screenshots` section and puts the block
above a trailer (a `---` rule or a "🤖 Generated with" line).

## Taking the screenshots

- **Shoot the real thing**: the dev server, a preview build, or the deployed site. Never a mockup.
- **Phone width (390×844) by default.** Add desktop (1280×900) when the layout differs there.
- **Before and after** for a visual fix, taken at the same size and scroll position.
- **Show the state the PR is about**: the empty state, the open menu, the error, the toggle
  switched on, not just the page's first screen.
- **Name files for what they show** (`after-phone-empty-cart.png`, not `shot1.png`); the name
  becomes the default caption. Pass `--caption` for anything the name can't say.
- **Look at every image yourself before uploading.** A screenshot of a spinner, a login wall or
  the wrong page is worse than none.
- **Check what's in the frame.** Uploaded images live in the repo, and in a public repo anyone can
  see them. No secrets or tokens, no personal data, no other customers' records, no notification
  banners. Use seed or demo data. Strip EXIF (GPS) from photos: re-saving through an image tool
  does it. Crop to the change.
- With the Playwright MCP, screenshots can only be written inside its allowed directories; save
  there and pass the paths to the script.

## Putting them in the PR

1. Open the PR first (the script needs its number), then run the script. It edits the body for you.
2. **Verify the images load in a logged-in browser.** `curl` is unauthenticated and `gh` doesn't
   fetch images, so neither proves it. Open the PR and check every image in the description
   rendered (in a browser automation tool: `naturalWidth > 0` on the body's `img` elements).
3. **Captions are by hand.** When a later change renumbers or moves what the screenshots show,
   re-shoot and re-run with `--replace`, then read each caption against its image.

For a comment or an issue, use `--print` and paste the markdown yourself (`--pr N` names the ref;
without a PR it uses `misc-<timestamp>`).

## Trade-offs and cleanup

- **Blobs are repo weight**, kept as long as the ref exists. Screenshots are small; videos are
  not, so keep clips short or link them from elsewhere. GitHub rejects files over 100 MB; the
  script stops at 50 MB.
- **Deleting a ref breaks that PR's images** once GitHub garbage-collects the commit. Usually
  just keep them. To remove one: `git push origin --delete refs/pr-screenshots/<pr>` (or the
  `screenshots/pr-<pr>` branch). List them with `git ls-remote origin 'refs/pr-screenshots/*'`.
- To fetch the images back (say, to reuse one in docs): `git fetch origin
  '+refs/pr-screenshots/*:refs/pr-screenshots/*'`, then `git show <sha>:<file> > out.png`.
- Don't push image files to the PR's own branch just to get a URL: they'd land in the diff and,
  after a merge, in the default branch forever.
