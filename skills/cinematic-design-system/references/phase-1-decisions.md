# Phase 1 — Decisions

**Output**: `docs/RESEARCH.md` (copy `assets/FILM_TEMPLATE/RESEARCH.md` and fill it in)

**Read at entry**: this file, `demo-uniqueness.md`, `anti-convergence.md`, `reference-protocol.md` (only if user provided references), and `data/directors-200.md`.

**Do not read at entry**: anything else. Phase 2 and Phase 3 data libraries are loaded in their own phases.

## Goal

Turn the user's Phase 0 inputs into a committed `(director, film, genre, niche, page list)` identity, a full research pass, a demo uniqueness audit, and a primary composition family — all written to `docs/RESEARCH.md`.

Phase 1 is blocked from starting until the Start Questionnaire is complete. See SKILL.md for the questionnaire sequence.

**Mode awareness**. The skill has two operating modes, and Phase 1 handles them differently:

- **Extraction mode** (`Screenshot`) — the reference is the subject. All five identity fields were *derived* in Phase 0 from the URL or image. Phase 1's job is to lock them, research deeper, and audit uniqueness against prior extractions.
- **Build modes** (`Step-by-step`, `Surprise me`) — the user is designing a new project. The identity fields come from the Phase 0 Entry Specification Rules. Phase 1's job is to research the chosen film, gate film selection with the 3-question test, and run the Demo Uniqueness Protocol against prior builds.

## Extraction Mode Flow

When the user provides a URL or screenshot in Phase 0:

### Step 1 — Fetch or inspect the reference

For a **URL**:
- Fetch the page HTML and extract `<title>`, H1, nav links, meta tags, visible palette, typography, footer.
- Follow a few nav links to discover the site's full page list. Stop at 4-5 pages; do not crawl the entire site.
- Note the path of the URL the user provided — the path segment is often the strongest hint about the reference's identity (e.g. `/demo/cloud-atlas`, `/projects/blade-runner`).

For an **image or screenshot**:
- Read it via multimodal inspection.
- Infer layout, palette, typography, and whatever content hints are visible.

### Step 2 — Derive the five identity fields

- **Film**: path segment first (`/demo/cloud-atlas` → *Cloud Atlas*), then visible content (poster-like hero images, title tag, meta descriptions), then aesthetic signature (palette + atmosphere + framing).
- **Director**: looked up from the film via web research or `data/directors-200.md`.
- **Genre**: implicit from the film.
- **Niche**: what kind of site is this? (Editorial magazine, architecture firm, record label, film portfolio, product launch, etc.) Infer from content, tagline, nav labels, footer.
- **Page list**: the actual pages discovered in the nav. Do not invent pages.

### Step 3 — Confirm with the user

Present the five derived fields in a single confirmation message, and ask the user to confirm or correct in one reply:

> *"I'll extract the design system describing this reference as: **{film}** ({director}) — **{niche}** — pages: **{page list}**. Proceed?"*

Accept corrections to any field. If the reference is genuinely ambiguous on a field (can't infer film from path or content, can't infer niche from content), ask for that field only. Do not re-run the whole questionnaire.

### Step 4 — Skip the niche/pages menu

Do **not** present the user with a list of niches or a list of page combinations. The reference IS the site. The identity fields are observations, not choices.

## Build Mode Flow (Step-by-step, Surprise me)

After the questionnaire closes, compute the committed `(director, film, genre)` triple using these rules. Never advance to research until the triple is determined.

| User supplied | Skill does |
|---|---|
| *film only* | **Derive** director and genre from the film. Web research when available; otherwise look up the film in `data/directors-200.md` via grep. |
| *director only* | Pick a film from the director's filmography via web research or `data/directors-200.md`. Record the implicit genre. |
| *genre only* | Pick a director from `data/directors-200.md` appropriate to the genre. Then pick a film from that director's filmography. |
| *director + genre* | Pick a film within that genre, ideally from the director's own filmography. If the director has never worked in that genre, use their lens against a genre-typical film and note the mismatch. |
| *director + film* | Use both as given. Allowed even if the director did not make that film — the director becomes the *lens*, the film becomes the *reference material*. Record the mismatch explicitly. |
| *genre + film* | **Disallowed.** The film already implies a genre. If both somehow arrive, keep the film and discard the genre with a one-line note in RESEARCH.md. The Start Questionnaire should block this combination at selection time. |
| *nothing* (Surprise me) | Pick a genre → director → film (or pick a film and derive director + genre from it). Honor the Demo Uniqueness Protocol shell-ban list when choosing. |

## Anti-Convergence: Three-Question Film Test

Answer these three questions from `anti-convergence.md`. The behavior differs by mode.

1. **What specific visual problem does this film solve for this niche?** Name a concrete cinematographic quality — not "it feels premium" or "it has a dark aesthetic". Example: *"Tarkovsky's Stalker uses extreme horizontal negative space and slow lateral reveals that match this architecture firm's need for spatial patience."*

2. **Would this same film work equally well for three unrelated niches?** If yes, the selection is too generic. A film that "works for any premium brand" is not a director's choice — it is a mood board.

3. **Are you picking the film or the film's reputation?** If the reasoning relies on reputation ("everyone knows this film is dark and sharp") rather than specific scenes, shots, or director decisions, that's association, not analysis. Rebuild justification from the film itself.

**In build modes**, these are a *gate*. If any answer is unsatisfactory, choose a different film before proceeding.

**In extraction mode**, the film is given by the reference — the skill is not selecting a film. The three questions become *descriptive analysis* of why the existing film works (or does not work) for the existing niche. Record the answers, but do not re-pick the film. If the answers reveal that the reference actually uses the film *poorly* for its niche, note that in `RESEARCH.md` as a finding — it is useful observation, not a blocker.

Record the three answers in `RESEARCH.md` under *Film → Why this film for this niche*.

## Web Research (required when web access is available)

When web access is available, research the committed director and film. This is **required, not optional**. Use primary/authoritative sources first, then secondary analysis to deepen interpretation.

**In both modes**, gather:

- Film palette and lighting behavior (dominant hues, accent strategy, contrast logic)
- Cinematography patterns (framing logic, camera behavior, scene rhythm, shot length)
- Production design and material cues (surfaces, textures, atmospheric layers)
- Director signature techniques (3 specific moves, not general reputation)

**In build modes**, also gather: 2-3 premium sites in the same niche as the user's project (to inform the shared system).

**In extraction mode**, also study the reference site itself in depth: scroll behavior, interactive states, actual color values (via devtools when possible), actual font stack, actual spacing scale. The reference site is the ground truth, and the film research is context that explains *why* the reference looks the way it does.

Record findings in `RESEARCH.md` under *Film Research Notes* and *Niche References*. Keep notes focused on structural signals. **Do not** paste plot summaries or trivia.

**If web access is unavailable**: say so explicitly in `RESEARCH.md`, mark the research pass as weaker, and continue with best-effort inference from `data/directors-200.md` and the skill's other data libraries.

## Reference Decomposition (build mode only)

In **extraction mode**, the reference IS the subject — there is no separate reference to decompose. Skip this section.

In **build modes**, if the user supplied a visual reference alongside their project spec (e.g. Step-by-step + an attached mood board URL), apply the protocol from `reference-protocol.md`:

1. **Classify by risk** — social platforms are high risk; brand/campaign sites are medium; editorial and film institutions are low.
2. **Extract only borrowable dimensions** — rhythm, density, typography attitude, image treatment, materiality, framing, interaction restraint, navigation posture.
3. **Forbid** full section order copying, hero composition recreation, grid skeleton reuse, color combination cloning without film justification.
4. **Rewrite** borrowed dimensions through the committed director and film.

Record the classification, borrowed dimensions, rejected dimensions, and rewriting plan in `RESEARCH.md` under *Reference Decomposition*. If no additional references were supplied (or if you are in extraction mode), delete that section from the filled-in RESEARCH.md.

**Watch for social-platform drift**: if a build-mode reference is Instagram/Pinterest/Behance/Medium-like, treat it as high risk. Extract only structural dimensions. Reject surface aesthetic entirely. See `anti-garbage.md` → Reference Drift.

## Demo Uniqueness Protocol

Run `demo-uniqueness.md` before closing Phase 1. Three artifacts must land in `RESEARCH.md`:

1. **Previous-work audit** — recurring shell traits from the user's prior outputs
2. **Shell-ban list** — layout traits forbidden for this project
3. **Primary composition family** — the committed structural direction

**In build modes**, the protocol works as originally described: the primary composition family must *differ* from the user's most recent comparable work, and the shell-ban list actively prevents the new build from inheriting prior shells.

**In extraction mode**, the framing shifts:

- The **primary composition family** is whatever the reference *already uses* — not something the skill picks. Identify it from the reference's layout and record it.
- The **previous-work audit** still surveys prior extractions in the same workspace.
- The **shell-ban list** checks whether this extraction is *meaningfully different* from prior extractions. If the user has already extracted the design system for a very similar cinematic site, this extraction may be redundant — flag that as a finding, don't fail it.
- Uniqueness is *diagnostic*, not a gate. The user may legitimately want to document two cinematic sites that happen to use similar shells.

If no prior outputs exist in either mode, still write a shell-ban list targeting common fallback templates.

## Sub-Agent Delegation

If the environment supports sub-agents, delegate these bounded Phase 1 tasks once the director + film are committed:

- Film research (palette, cinematography, signature techniques)
- Niche research (2-3 premium sites in the same niche)
- Reference decomposition (for each user-supplied reference)

The lead agent retains responsibility for:

- Committing the `(director, film, genre)` triple
- Writing the Demo Uniqueness Audit
- Approving the research pass before Phase 2 begins

Do not let multiple agents independently redefine the director or film.

## Phase 1 Completion Gate

Phase 2 is blocked until `docs/RESEARCH.md` has all of the following:

- [ ] Entry mode recorded (`Screenshot` / `Step-by-step` / `Surprise me`)
- [ ] Director, film, genre, niche, pages all committed (derived from the reference in extraction mode; from the questionnaire in build modes)
- [ ] Anti-convergence 3-question test answered (gate in build modes; descriptive in extraction mode)
- [ ] Film research notes (or explicit "web access unavailable" marker)
- [ ] In build modes: 2-3 niche references (when web access available). In extraction mode: in-depth study of the reference site itself.
- [ ] In build modes: Reference decomposition section filled in or deleted. In extraction mode: section deleted.
- [ ] Previous-work audit
- [ ] Shell-ban list (gate in build modes; diagnostic in extraction mode)
- [ ] Primary composition family (chosen in build modes; observed in extraction mode)
- [ ] Research pass quality marked (strong / adequate / weak)

When all boxes are checked, close Phase 1 and move to `phase-2-storyboard.md`.
