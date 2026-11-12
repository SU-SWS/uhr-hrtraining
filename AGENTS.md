# AGENTS.md

Guidance for AI coding agents (and humans) working in this repository.

## What this is

A static site of Stanford University Human Resources (UHR) e-learning courses,
served at **https://hrtraining.stanford.edu/** via GitHub Pages. There is no
application code: the root `index.html` is a hand-written landing page that
links to each course, and every other top-level folder is a self-contained
course exported from an authoring tool.

The site is intentionally hidden from search engines (`robots.txt` disallows
all, and pages carry `noindex`). Keep it that way.

## Layout

```
index.html               Landing page: list of courses (hand-edited)
styles/styles.css        Tailwind entry for the landing page
styles/centered.tsx      Tailwind plugin providing the `.centered` container
tailwind.config.js       Tailwind config, using Stanford's Decanter theme
css/                     Build output (git-ignored, built in CI)
scss/                    Unused leftover
.github/workflows/deploy.yml  Build + deploy to GitHub Pages
<course-folder>/         One exported course per folder (see below)
```

### Courses and their source tools

| Folder | Course | Exported from |
|---|---|---|
| `intro-to-sequoia/` | Introduction to Sequoia | Articulate Storyline (SCORM package; `story_content/`, `imsmanifest.xml`) |
| `welcome-to-sequoia/` | Welcome to Sequoia | Articulate Rise (`lib/`, `runtime-data.js`) |
| `foundations-of-compensation/` | Foundations of Compensation | Articulate Rise |
| `ta-hiring-managers/` | Talent Acquisition for Hiring Managers | Genially (offline HTML export) |
| `pm-employees-*` (5) | Performance Management for Employees | Genially |
| `pm-managers-*` (5) | Performance Management for HR and Managers | Genially |

The five PM topics, for both audiences, are: `understanding-edge`,
`goal-setting`, `communication-feedback`, `development-planning`,
`performance-conversations`.

## Working with course exports

Treat course folders as **generated vendor output**. Don't reformat, lint, or
refactor them; changes to content belong in the authoring tool, followed by a
fresh export.

### Adding a new course

1. Put the export in a new top-level folder with a short, lowercase,
   hyphenated, URL-safe name (no spaces, parentheses, or underscores), e.g.
   `pm-employees-goal-setting`.
2. Make sure the course's entry page is `index.html` so the folder URL works.
   Genially exports name it `genially.html`; rename it.
3. Delete export boilerplate that isn't part of the course, such as Genially's
   `Guide-to-presenting-offline.pdf`.
4. Add a link to the root `index.html` course list, using the absolute
   `https://hrtraining.stanford.edu/<folder>/` form the other links use.
5. Optionally optimize media (see below).

### Genially specifics (important)

- The course data is stored **base64-encoded** in `index.html`
  (`window.dataBase64="..."`), so a plain grep **won't** find asset paths.
  Audio, for example, is played from `./audios/<id>.<ext>` paths inside that
  blob. The hidden `<audio><source>` tags near the end of `index.html` only
  preload the files.
- To rename or re-encode an asset, decode the blob, replace the paths
  byte-for-byte, re-encode it, **and** update the `<source>`/`<img>` tags.
  Then load the course in a browser and check that the asset still works.
- Image filenames are referenced by exact name, so keep the name and
  extension when compressing.
- `static/js/*.js` is Genially's bundled player. It contains public client IDs
  (Firebase web API key, Sentry DSN, analytics IDs) that trip gitleaks (see
  Committing).

### Media optimization (what was done, and how)

- Images: ImageMagick (`convert`). Lossless PNG recompression for photos;
  256-color dithered quantization only when the result stays visually
  identical (PSNR ≥ 40 dB); JPEG at quality 82. Keep a file only if it got
  smaller, and keep the filename.
- Audio: `ffmpeg -i in.wav -ac 1 -c:a libmp3lame -b:a 64k out.mp3`, which works
  well for voice narration. Requires the Genially reference update described
  above.
- Many images repeat across the PM courses, so optimize by content hash and
  copy each result to every copy.

## Landing page styles

Only the root `index.html` uses Tailwind. Its `content` glob is `./index.html`
only, so course folders aren't scanned.

```bash
corepack enable
yarn install
yarn build-css    # styles/styles.css -> css/styles.css
yarn watch        # rebuild on change
```

Stack: Tailwind 3 plus the [Decanter](https://github.com/SU-SWS/decanter)
theme (Stanford's design system), Yarn 4 (`packageManager` in
`package.json`).

## Deployment

`.github/workflows/deploy.yml` runs on pushes to `main`. It installs with
`yarn install --immutable`, builds the CSS, and uploads the whole repo root as
the Pages artifact. Dotfiles are excluded automatically. Pull requests only
build; they never deploy. A manual run (`workflow_dispatch`) redeploys.

- Because the artifact is the repo root, **anything committed is published**.
  Don't commit drafts, source files, or anything private.
- If you change dependencies, commit the updated `yarn.lock`, or CI fails.

## Git gotchas

- `.gitignore` entries without a leading `/` match at **any depth**. A bare
  `css` once silently excluded every course's `css/` folder and broke styling
  in production. Anchor root-only ignores (`/css`), and after adding a course
  check `git status --ignored` to confirm nothing in it is ignored.
- The maintainer's global pre-commit hook runs gitleaks. Genially player
  bundles produce known false positives in `*/static/js/`. Skip the scan for a
  single commit with `SKIP_GITLEAKS=1 git commit ...` only after confirming the
  findings are in vendor bundles, or add a repo `.gitleaks.toml` allowlist for
  that path.
- GitHub Pages has a 1 GB site limit, and git warns about files over 50 MB.
  Compress large media before committing.

## Local preview

Serve the repo root over HTTP. Course players don't work from `file://`.

```bash
python3 -m http.server 8000
# http://localhost:8000/<course-folder>/
```

Landing-page links are absolute production URLs, so use the course folder URLs
directly when testing locally.
