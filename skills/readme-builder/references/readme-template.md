# README Template

This reference is loaded during Phase 5 of the readme-builder workflow. It defines the canonical section order, the heading synonyms recognized when merging an existing README, and the per-section drafting rules.

The goal is consistency *across* projects, not a rigid mold *per* project. Skip sections that don't apply. Keep prose short. Use code blocks for commands. Avoid filler.

## Table of Contents

1. [Section order](#section-order)
2. [Canonical heading synonyms](#canonical-heading-synonyms)
3. [Per-section drafting rules](#per-section-drafting-rules)
4. [Example output block — small web app](#example-output-block--small-web-app)
5. [Example output block — Python library](#example-output-block--python-library)
6. [Voice and style](#voice-and-style)

---

## Section order

The canonical order (applicable sections are selected from the type matrix in `project-types.md`):

1. Title + tagline (no heading; the H1 IS the title)
2. Badges (optional; if existing README has them and they look maintained, preserve)
3. What it does
4. Screenshots
5. Quick start
6. Development
7. Environment variables
8. Architecture
9. Usage
10. API reference
11. Testing
12. Deployment
13. Troubleshooting
14. Contributing
15. License

Custom sections from the existing README (e.g., *Sponsors*, *Roadmap*, *Acknowledgements*) slot in at their original relative position. If a custom section appeared between *Testing* and *Deployment* in the old README, keep it there.

---

## Canonical heading synonyms

When parsing an existing README in Phase 2, treat these synonyms as the same canonical section. Headings are matched case-insensitively and ignoring trailing punctuation.

| Canonical | Synonyms also recognized |
| --- | --- |
| What it does | About, Overview, Introduction, Description, Summary |
| Screenshots | Screenshot, Demo, Preview, Gallery |
| Quick start | Quickstart, Getting started, Installation, Install, Setup, Set up |
| Development | Develop, Dev, Local development, Local setup, Development setup |
| Environment variables | Env vars, Environment, Configuration, Config, Env |
| Architecture | Project structure, Layout, Structure, Codebase layout |
| Usage | How to use, Example, Examples, Usage examples, Basic usage |
| API reference | API, Reference, Public API |
| Testing | Tests, Running tests, Test |
| Deployment | Deploy, Hosting, Release, Publishing, Publish |
| Troubleshooting | FAQ if it's clearly debug-style, Common issues, Known issues, Gotchas |
| Contributing | Contributions, Contribute, How to contribute |
| License | Licence, Licensing |

Anything that doesn't match a canonical heading or a synonym is treated as a custom section and preserved verbatim.

---

## Per-section drafting rules

### Title + tagline

```markdown
# <project-name>

<one-sentence tagline>
```

- `<project-name>` from manifest `name` field. Fall back to repo directory name.
- Tagline: one sentence (15–20 words max) describing what the project is and who it's for. Derive from manifest `description` field; if absent, infer from the codebase carefully — and if you can't, `TODO: write a one-sentence project tagline.`
- Do not include "A `<word>` for `<word>`" boilerplate. Be specific.

**Good**:
> A CLI for batching FFmpeg conversions across folders, with resume support for interrupted runs.

**Bad** (vague):
> A modern, fast, lightweight tool for working with video files.

### Badges

If the existing README has badges (npm version, CI status, license, etc.) preserve them. Don't auto-generate new ones in v1 — fabricating a badge URL points to nothing. (Future: add badge auto-detection.)

### What it does

2–4 sentences. Answers:
1. What problem does it solve?
2. Who is it for?
3. What's the most distinctive thing about it?

Optionally followed by a short bullet list of headline features (3–6 bullets max). Each bullet a one-line user-visible capability — not implementation details.

If you can't write this confidently from code: `TODO: write a 2–4 sentence description of what this project does and who it's for.`

### Screenshots

```markdown
## Screenshots

![Home page](docs/images/home.png)

![Dashboard](docs/images/dashboard.png)
```

- One image per captured route, in order.
- Alt text is a short label (the route's user-facing name, not the path).
- If screenshots phase failed or was skipped, the section either is omitted entirely (for non-UI projects) or contains a single TODO line explaining the situation.

### Quick start

Step-by-step commands for the fastest path from cloned repo to running code. Use a numbered list, with code blocks per step.

```markdown
## Quick start

1. Clone the repo:
   ```bash
   git clone <repo-url>
   cd <repo-name>
   ```
2. Install dependencies:
   ```bash
   <package-manager-install-command>
   ```
3. (If applicable) Configure environment:
   ```bash
   cp .env.example .env
   # then edit .env with your values
   ```
4. Start the dev server:
   ```bash
   <dev-command>
   ```

The app will be available at <url>.
```

- Use the package manager you detected from the lockfile (`npm`, `pnpm`, `yarn`, `bun`, `pip`, `uv`, `poetry`, `cargo`, `go`, `bundle`).
- Step 3 only if `.env.example` or equivalent exists.
- Step 4 only if there's actually a dev command for this project type. For a library, replace with the install command and a one-line import example.
- The "available at" line only for `web-app-browser` and `api-service` types where you know the port.

### Development

A flat list of common commands beyond the basic dev server. Format as a table or a labeled list:

```markdown
## Development

| Command | Description |
| --- | --- |
| `pnpm dev` | Start the dev server with HMR |
| `pnpm build` | Build for production |
| `pnpm lint` | Run ESLint |
| `pnpm typecheck` | Run TypeScript without emitting |
```

Include only commands actually present in `package.json#scripts` / `Makefile` / `pyproject.toml` / equivalent. Skip the section if the only commands are install + dev (those are already in Quick start).

### Environment variables

Only if env vars were discovered. Format as a table:

```markdown
## Environment variables

Copy `.env.example` to `.env` and set the following:

| Variable | Required | Description |
| --- | --- | --- |
| `DATABASE_URL` | Yes | Postgres connection string |
| `STRIPE_SECRET_KEY` | If billing enabled | Stripe API key |
| `LOG_LEVEL` | No (default: `info`) | One of `debug`, `info`, `warn`, `error` |
```

- "Required" is best-guess. If `.env.example` shows a placeholder value and the variable is referenced with no fallback, it's likely required. If there's a fallback (`process.env.X ?? "default"`), it's optional — note the default.
- "Description" from a comment near the reference, or from `.env.example` comments. If nothing's available: `TODO: describe`.

### Architecture

A short prose paragraph (3–5 sentences) describing the layout, followed by a directory bullet list.

```markdown
## Architecture

The codebase is a [framework] app with [database] for persistence and [auth solution] for authentication. UI components live in `components/`, page routes in `app/`, and database queries are centralized in `lib/db/`.

- `app/` — route definitions and page components (Next.js App Router)
- `components/` — reusable UI components
- `lib/` — utilities, API clients, database queries
- `prisma/` — database schema and migrations
- `public/` — static assets
```

- Name the framework, persistence layer, and auth solution only if you actually identified them.
- The bullet list covers top-level directories that a contributor would touch. Skip directories that are just generated output (`dist/`, `build/`, `node_modules/`).
- For libraries, prefer a different structure: skip this section in favor of API documentation.

### Usage

For libraries and CLIs. Show 2–3 minimal examples that exercise the headline features.

For a library:

````markdown
## Usage

```ts
import { foo, bar } from "<package-name>";

const result = foo("hello");
console.log(bar(result));
```
````

For a CLI:

````markdown
## Usage

```bash
# Convert a single file
<cli-name> convert input.mp4 --output output.mkv

# Process a folder
<cli-name> batch ./videos --output-dir ./converted
```
````

- Examples must be plausible — derive them from public exports (libraries) or registered subcommands (CLIs).
- Do not over-document. Three examples is plenty; a full reference belongs in dedicated docs.

### API reference

For libraries with a clearly enumerable public API, or APIs with route definitions.

For a library, list exports with one-line descriptions:

```markdown
## API

### `foo(input: string): string`
Transforms `input` into <description>.

### `bar(input: string): number`
Counts <something> in `input`.
```

For an API service, list endpoints grouped by resource:

```markdown
## API

### Posts

- `GET /api/posts` — list posts (paginated)
- `POST /api/posts` — create a post
- `GET /api/posts/:id` — fetch a single post
- `PATCH /api/posts/:id` — update a post
- `DELETE /api/posts/:id` — delete a post
```

For larger APIs (>10 endpoints), put detailed reference in a separate file (`docs/API.md`) and link to it from the README.

### Testing

```markdown
## Testing

Run the test suite:

```bash
<test-command>
```
```

- Use the test command identified in Phase 3.
- If there are multiple test types (unit + e2e), list both with their respective commands.
- Skip the section entirely if no test setup was detected.

### Deployment

This section requires evidence of a deployment setup. Examples:

```markdown
## Deployment

This project deploys to Vercel. Pushes to `main` deploy to production automatically via the GitHub integration.

To deploy manually:

```bash
vercel --prod
```
```

```markdown
## Deployment

The project ships as a Docker image. Build and run:

```bash
docker build -t <name> .
docker run -p 3000:3000 <name>
```

For production, see `docker-compose.yml`.
```

- Only describe deployment paths the repo actually supports.
- If no deployment config exists, **omit the section** for libraries / CLIs (deployment doesn't apply) and write a TODO for web apps / APIs / native apps.

### Troubleshooting

Always a TODO placeholder unless the existing README had a Troubleshooting section, in which case preserve it.

```markdown
## Troubleshooting

TODO: add common issues and their fixes. Suggested entries: development environment setup, dependency conflicts, runtime errors, platform-specific gotchas.
```

(Troubleshooting content emerges from actual user pain — autopopulating it leads to fabricated advice. Always TODO this on first pass.)

### Contributing

```markdown
## Contributing

Contributions welcome. See [CONTRIBUTING.md](./CONTRIBUTING.md) for the development workflow, coding standards, and how to open a pull request.
```

- If `CONTRIBUTING.md` exists, link to it.
- If not, write a short paragraph: "Open an issue to discuss before submitting a PR. Run tests with `<test-command>` before opening a PR."
- For closed-source / personal projects, skip the section entirely.

### License

```markdown
## License

<license-type> — see [LICENSE](./LICENSE) for details.
```

- License type from the LICENSE file header (MIT, Apache-2.0, GPL-3.0, etc.).
- If no LICENSE file: skip the section. Do not invent a license.

---

## Example output block — small web app

What a fresh README for a small Next.js app might look like end-to-end. This is illustrative, not a template to copy literally.

````markdown
# Threadbook

A self-hosted archive viewer for your Threads (Meta) posts, with backfill, full-text search, and CSV export.

## Screenshots

![Home](docs/images/home.png)

![Search](docs/images/search.png)

![Settings](docs/images/settings.png)

## Quick start

1. Clone:
   ```bash
   git clone https://github.com/you/threadbook
   cd threadbook
   ```
2. Install:
   ```bash
   pnpm install
   ```
3. Configure:
   ```bash
   cp .env.example .env
   # set THREADS_ACCESS_TOKEN and DATABASE_URL
   ```
4. Run migrations and start:
   ```bash
   pnpm db:migrate
   pnpm dev
   ```

App available at http://localhost:3000.

## Development

| Command | Description |
| --- | --- |
| `pnpm dev` | Dev server with HMR |
| `pnpm build` | Production build |
| `pnpm db:migrate` | Apply database migrations |
| `pnpm lint` | ESLint |

## Environment variables

| Variable | Required | Description |
| --- | --- | --- |
| `DATABASE_URL` | Yes | Postgres connection string |
| `THREADS_ACCESS_TOKEN` | Yes | API token from Meta Threads |
| `NEXT_PUBLIC_APP_URL` | No (default: `http://localhost:3000`) | Public URL for OAuth callbacks |

## Architecture

A Next.js App Router app with Postgres (via Drizzle ORM) for storage and Threads OAuth for auth.

- `app/` — routes and page components
- `components/` — UI components
- `lib/db/` — Drizzle schema and queries
- `lib/threads/` — Threads API client
- `drizzle/` — migrations

## Testing

```bash
pnpm test
```

## Deployment

Configured for Vercel. Pushes to `main` deploy automatically.

## Troubleshooting

TODO: add common issues and their fixes.

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md).

## License

MIT — see [LICENSE](./LICENSE).
````

---

## Example output block — Python library

````markdown
# slugify-py

A small, dependency-free Python library for converting strings to URL-safe slugs.

## Quick start

```bash
pip install slugify-py
```

```python
from slugify_py import slugify

slugify("Hello, World!")           # "hello-world"
slugify("café & croissants", lower=False)  # "cafe-croissants"
```

## Usage

```python
from slugify_py import slugify

# Default: lowercase, hyphen-separated
slugify("Some String")  # "some-string"

# Custom separator
slugify("Some String", separator="_")  # "some_string"

# Preserve casing
slugify("Some String", lower=False)  # "Some-String"
```

## API

### `slugify(text: str, *, separator: str = "-", lower: bool = True) -> str`
Convert `text` to a URL-safe slug.

- `text` — the input string.
- `separator` — character used to join words (default `-`).
- `lower` — lowercase the output (default `True`).

## Testing

```bash
pytest
```

## Contributing

Open an issue to discuss changes before submitting a PR. Run `pytest` before opening a PR.

## License

MIT — see [LICENSE](./LICENSE).
````

---

## Voice and style

- **Active voice, present tense.** "Threadbook archives your posts" — not "Threads can be archived by Threadbook" and not "Threadbook will archive your posts".
- **Second person for instructions.** "Run this command", not "One should run this command".
- **Sentence case for headings.** "Quick start", not "Quick Start" or "QUICK START".
- **Code blocks for commands.** Inline code (backticks) for variables, file paths, short identifiers.
- **No filler words.** "Simply", "just", "easy" — strike these. If something is easy, the steps will show it.
- **Concrete numbers.** "30s timeout", not "a short timeout". "3000", not "the default port".
- **Don't promise.** "Coming soon", "planned", "future work" — these belong in a CHANGELOG or ROADMAP, not the README.
- **TODOs are loud.** Use `TODO:` prefix; leave them where a reader will see them.
