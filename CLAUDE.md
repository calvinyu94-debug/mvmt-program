# MVMT Program — working notes

Printable home-exercise program builder for clinical use. Single self-contained
`index.html` at repo root, served by GitHub Pages from `main`.

## Hard constraints

- **No backend, no auth, no build step.** Everything client-side, one file.
  Google Fonts via CDN is the only external dependency, and the page must stay
  legible if it fails to load.
- **No patient data in the repo, ever.** The Pages site is public. `.gitignore`
  blocks `*.json` for this reason — there is no legitimate `.json` to track.
- **Print fidelity is a feature.** The printed handout is the actual deliverable
  to patients. Preserve and extend the `@media print` rules.
- **Colour lives in `:root`, nowhere else.** See the theme layer below.
- **The library is user-mutable. Nothing may assume it is fixed at boot.** Any
  check, assertion or resolution over `EX` must run on **library change**, not
  once at boot. Boot-time-only logic misses clashes and resolutions introduced
  by custom entries, and misses their removal.

  `rebuildLibrary()` is that hook, and it is also what runs at boot — so putting
  the work there gets the boot pass for free and cannot drift out of step. Two
  things live there today for exactly this reason: `resolveAnatomy()`'s
  name-to-id lookup (a custom entry can satisfy a name the map needs, and
  deleting one takes it away again) and the exercise-name uniqueness assertion
  (a custom entry can introduce a clash long after load). This has been got
  wrong twice; assume the next one too, and check before writing `// at load`.

## `alsoRegion` is coupled to a safety callout — read before cross-tagging

`alsoRegion` exists for **discoverability**: it cross-lists an exercise under one
or more further region filters without changing its `id`, so no migration is
needed. It takes a string or an array — the ten scapular drills filed under
`region: "thoracic"` carry `alsoRegion: "shoulder"`, and single leg balance is
`["knee","ankle-foot"]` on top of its hip primary.

**But it is not purely cosmetic.** The patient handout renders the neck safety
callout whenever a phase contains any exercise that reports `"cervical"` from
`regionsOf()` — which reads `alsoRegion` as well as `region` — and the jaw
safety callout on `"head-jaw"` by exactly the same test. So:

> Cross-tagging an exercise into `"cervical"` or `"head-jaw"` silently attaches
> that region's safety callout to every handout that contains it.

That widening is deliberate — for a safety callout, firing on any involvement is
the correct failure direction. The consequence is that `alsoRegion: "cervical"`
and `alsoRegion: "head-jaw"` are **clinical** decisions, not filing ones.

**Never cross-tag into either for filter convenience alone.** If an exercise
should appear under the Cervical or Head & jaw filter but genuinely does not
warrant those warnings, the callout logic needs to change first — do not tag and
hope.

Cross-tagging into the other seven regions carries no such side-effect today. If
a further region gains its own forced callout, this note needs updating
alongside it.

The manual-entry form exposes `alsoRegion` as a dropdown, so this decision is now
made by whoever is typing rather than by whoever is editing the library. That is
why the field carries an inline note naming the consequence. **Keep that note
truthful if the callout rules change.**

## Exercise schema

```js
{
  id: "hip-mob-9090-switch",   // {region}-{mob|stab|nrv}-{name-slug}, stable
  region: "hip",               // head-jaw|cervical|shoulder|elbow-wrist|thoracic
                               // |lumbar|hip|knee|ankle-foot
  alsoRegion: "shoulder",      // optional, string or array — see the coupling warning above
  type: "mobility",            // mobility | stability | nerve
  level: 1,                    // 1 Foundational | 2 Intermediate | 3 Advanced
  name: "90/90 Hip Switch",   // MUST BE UNIQUE — it is a resolution key, see below
  desc: "...",                 // clinical copy handed to patients — do not reword
  targets: "...",              // likewise
  rx: "2 x 8 each side, slow"  // default prescription; every entry must have one
}
```

`desc` and `targets` are written to be read by patients. Do not rewrite,
summarise or "improve" them without being asked.

### `name` is a resolution key, not just a label

The whole `ANATOMY` map points at exercises **by name**. `resolveAnatomy()` takes
the first match and reports success, so a second exercise under an existing name
does not collide loudly — it does something worse:

> Adding a drill whose name already exists makes it **unreachable by the map**,
> and silently hands every reference that meant it a different exercise.

This has happened once. A shoulder *Wall Ball Circles* and a cervical one were
written in separate batches; nothing broke, because no structure happened to
reference the name yet. The cervical one is now **Wall Ball Head Circles**. The
next collision would not necessarily be found by luck, so it is asserted instead.

`resolveAnatomy()` groups every entry in `ALL` by the same
case-and-whitespace-insensitive key its own fallback lookup uses — two names
differing only in capitalisation are equally indistinguishable to it — and
reports any name held by more than one entry to the console, to the Anatomy
banner, and, for each structure that referenced an ambiguous name, on the
structure itself. It runs on every library change, not once at boot, because a
**custom entry can introduce a clash long after load** and deleting it takes the
clash away again.

Structure ids are cheaper still, and Batch D established how far: `ANATOMY` ids
are referenced only by `inherits`, and **no saved program stores one** — every
item a program holds is `{exerciseId, rx, freq, note}`, from all four sites that
build one. A structure can therefore be retired outright, which is what happened
to `thoracic-erector-spinae` and `lumbar-erector-spinae` when the nine named
columns arrived: a structure cannot be both "the thoracic part of all three
columns" and "the whole of one column", so the two cuts could not coexist.
**Verify the absence of references before deleting rather than trusting this
paragraph** — it is a fact about today's code, not a guarantee.

Renaming an exercise is cheap in a way that renaming an id is not: saved programs
store `exerciseId`, so a `name` change costs nothing downstream. The one thing it
touches is any `ANATOMY` `ex` entry naming it — and that will report itself
immediately if you forget. **Fix a clash by renaming, never by leaving both.**

User-authored entries use this same shape and are indistinguishable to every
renderer. They carry four fields on top of it — `custom: true`, the raw
`description` and `progression` the form was filled in with, and `createdAt` /
`updatedAt`. `desc` is stored **assembled** from the first two, so nothing
downstream has to know the difference. See the custom-exercise section below.

### "Tennis elbow" is a deliberate, approved exception

The tool was rebranded away from its origin as a return-to-tennis builder, and
one acceptance check was written as *"no occurrence of 'tennis' in any UI-facing
string"*. Every surviving occurrence is deliberate. Three are in elbow-wrist
`targets`:

- "the group involved in tennis elbow"
- "the reliable starting point for a painful tennis elbow"
- "Tendon loading for tennis elbow"

three arrived with the anatomy index, in `ANATOMY` `action` / `clinical`:

- `elbow-common-extensor` — "Tennis elbow. Isometrics first, eccentrics second…"
- `elbow-ecrb` — "the primary tendon involved in tennis elbow"
- `nerve-radial` — "Can mimic tennis elbow closely"

and three more with Batch C, all in `clinical` and all diagnostic — two of them
exist precisely to say *this is not tennis elbow*:

- `elbow-wrist-ecrl` — "why some people with tennis elbow hurt more carrying a bag"
- `elbow-wrist-anconeus` — "frequently tender in tennis elbow and mistaken for the extensor origin"
- `elbow-wrist-supinator` — "outer-elbow pain that mimics tennis elbow but sits a few centimetres further down"

**Do not "fix" any of these.** Tennis elbow is lateral epicondylitis — the
condition name patients actually use. It is clinical vocabulary, not sport
framing. The acceptance check was written too literally; the rule it was
protecting is that nothing frames the tool around a sport.

The check as originally phrased — *exactly three hits, all in `targets`* — was
stale within one batch, and its literal form was already the thing that made it
wrong. **Do not restate it as a count.** It has now been "exactly three", "six"
and "nine", and it will keep moving: the differential around lateral elbow pain
is most of what this region is for, so every forearm batch adds hits.

State it against the rule instead: **every hit must be the condition name, and
none may frame the tool around the sport.** A hit that reads as diagnosis passes
wherever it sits. A hit that reads as sport fails wherever it sits.

Every `id` must start with its own `region` key. Ids are referenced by saved
programs, so changing one costs a `LEGACY_ID_MAP` entry — get them right before
content ships, not after.

## Adding a region

Region and type filter chips, the library groupings and the colour legend are all
derived from what the library actually contains, so a new region is a content drop
plus two small declarations. There is **no schema change and no migration** — the
`region` field is a free string and `migrate()` does not validate it.

1. Add `{k, name}` to `REGIONS` (order in that array is the display order).
2. Add a `--r-<key>` colour token, plus `.ex.<key>` and `.acard.<key>` rules.
   The region colours are a scanning aid across a 311-entry list, so a tenth one
   has to stay distinguishable from the other nine **and** from `--accent` —
   see the theme section. Note that the eight chromatic values already cover the
   usable hue wheel and `--r-head-jaw` has spent the one "outside the system"
   slot, so a tenth region is a genuinely hard colour problem, not a lookup.
3. Ship the content, with ids prefixed `<key>-`.

The filter chip, the library grouping and the legend entry then appear on their
own. A region declared with no content shows nothing at all rather than an empty
filter, so step 1 can safely land ahead of step 3.

If the new region needs a forced safety callout, that is a separate change — see
the callout section below and the `alsoRegion` warning above.

## The theme layer

Every colour, radius, shadow and font family the page uses is declared on
`:root` and referenced through `var()`. **Nothing below that block should carry
a literal colour.** The check is one line, and it should come back empty:

```
strip the /* */ comments and the :root{...} blocks from the <style>, then grep
for '#rrggbb' or 'rgba(' — anything left is a colour that escaped the theme
```

The only other place literals appear is the second `:root` block inside
`@media print`, which re-points the same tokens at flat white. Print is a token
swap, deliberately, rather than a pile of overrides scattered through the rules.

### Semantic states must never alias a categorical token — read before any palette change

Same shape as the `alsoRegion` coupling above, and it will recur on the next
theme change.

`--r-lumbar` was serving as the danger colour. That was invisible while the
palette happened to be orange, and wrong the moment it wasn't: the Clay Soft
swap turned it green, and a green Delete hover and green validation errors would
have shipped. The affected rules were `.rm:hover`, `.pbtn.danger`,
`.ex-acts button.danger` and `.modal-body .ferr`; they are on `--danger` now.
The phase-level colours on the tabs (`.st1/.st2/.st3`) had the same defect,
aliased onto `--r-hip` / `--r-lumbar` / `--r-thoracic` via `--hip/--lum/--tsp`;
they are on explicit `--level-1/2/3`, ordered cool to warm.

> **Before any palette change, grep every colour token for uses outside its
> named category.** A `--r-*` token outside a `.ex.<region>` / `.acard.<region>`
> rule or the legend is a coupling, not a colour choice.

The region palette is chosen for mutual distinctness, not for meaning. Nothing
that carries meaning — danger, warning, success, level — may point at it.

### The ninth region colour is deliberately not a colour

Eight muted hues cover the usable wheel. The ninth region, `head-jaw`, is
`#3E3630` — a warm near-black — because the only hue-space left was beside
cervical rose, and head-jaw and cervical are the pair most often compared: they
are anatomically adjacent and sit next to each other in `REGIONS`. A ninth hue
would have collided exactly where the scanning aid is needed most.

Reading it as "the head is outside the spine-and-limb colour system" is the point
rather than a rationalisation after the fact. It also means **the trick does not
generalise**: there is one non-chromatic slot and it is taken. A tenth region has
to either find real hue space or force a repaint of the eight.

### Two more a theme swap is not allowed to forget

**Shadows do not print.** The panels carry their structure with `--shadow` and
no border, so `@media print` puts a real `1px solid var(--line)` back on
`.panel`, and states `.callout`'s border rather than letting it inherit one that
the tinted ground was helping. Radii are zeroed there too: a 16px corner on a
hairline prints as four broken corners on some drivers. If you move a boxed
element onto elevation, give it a print rule in the same change.

**`.sheet` takes no border in print, deliberately.** It is the only boxed element
that does not get its outline restored, because on paper the sheet *is* the page
— the `@page` margin box is its border, and drawing another one boxes the
handout inside the margin. On screen it carries `--shadow` like any other panel.
Do not "fix" the asymmetry.

**The chrome is light.** The top bar was built for a dark ground — white text on
`rgba(255,255,255,…)` borders — which is invisible on a light one. Bar controls
are on `--chrome`, `--chrome-fg`, `--chrome-dim` and `--chrome-btn`; if the bar
ever inverts back, those four tokens are the whole job.

### The two muted tokens have a contrast floor

`--ink-3` and `--chrome-dim` are the tokens a theme reaches for when something
should recede. Both are already at their floor: each is the **least-shifted
colour that clears 4.5:1 against every ground it sits on**, and neither has room
to be lightened again.

`--ink-3` carries `What it's for:` on the printed handout — read by patients in
pain, often older, rarely in good light. Its grounds are `--panel`, `--panel-2`,
`--accent-wash` and `--hover`; the binding one is a safety callout's heading on
`--accent-wash`, which is tighter than white and is the case a white-only check
misses.

`--chrome-dim` carries the **unselected** view-switch labels. The control someone
is about to click is by definition the unselected one, so it is a bad place for
text that is merely almost readable. Its only ground is `--chrome`.

**Check them against the real computed ancestor background, not the ground you
assume is in play.** Walk up from the element to the first non-transparent
background and measure against that — assuming white is how the callout-heading
case got missed once already.

The cost of `--ink-3` sitting at its floor is that the third step of the ink ramp
is 11.1 L* where the first two are ~21.7. That compression is bought
deliberately. Do not lighten it back for the sake of an even ramp; if the ramp
needs evening out, move `--ink-2`.

### Fonts

`--font-display` (Fraunces) is reserved for the brand mark, the handout's
patient name and the exercise numbers. Everything else is `--font-body` /
`--font-ui` (DM Sans). Both carry real fallbacks, because the page has to stay
legible when the CDN doesn't answer.

## Forced handout callouts

All three are non-dismissible and render automatically. Where a phase triggers
more than one, they print **neck, then jaw, then nerve** — the order comes from
the render sequence in `renderPatient()`, so keep
`${cervicalCallout}${jawCallout}${nerveCallout}` in that order.

| Trigger | Callout |
|---|---|
| any exercise with `"cervical"` in `regionsOf()` | About the neck exercises |
| any exercise with `"head-jaw"` in `regionsOf()` | About the jaw exercises |
| any exercise with `type: "nerve"` | About the nerve glides |

A cervical nerve glide raises **both** of the applicable ones, which is correct:
it is cervical by region and a nerve glide by type, so both sets of rules apply.

**The bar for a fourth is high, and it should stay high.** Callouts on everything
train people to read none of them. Jaw cleared it on one argument: a patient can
lock their jaw open, which is an emergency-department visit rather than a bad
week. A callout that only warns against overdoing it does not clear the bar —
that belongs in the exercise's own `desc`.

All three render as `<div class="callout safety">`. The therapist's note and the
closing guidance are plain `callout`s. The difference is a heavier left rule and
the accent wash — **weight, not hue**, so it survives a mono clinic printer,
where the wash flattens to white and the rule is all that is left. If you add a
callout, decide which of the two it is.

## The closing guidance must stay true of the sheet it prints on

Every line of "How to run this program" is conditional, because each one asserts
something about the sheet's contents. This was found the hard way: a sheet of two
nerve glides printed *"Move to the next stage once the current one feels
controlled"* a few centimetres below a safety callout reading *"Fewer is better.
A handful of slow repetitions is a full dose."* The guidance was arguing with the
warning above it.

| Line | Prints when |
|---|---|
| Mobility first, then strength | phase has **both** mobility and stability work |
| Where you should feel it | always |
| Progressing | at least one prescribed exercise has a progression |
| Consistency beats intensity | always |
| the whole block | phase has at least one exercise |

The same rule applies to section headings — `Part 1 — Mobility (do these first)`
drops its qualifier when there is no second part to be before.

**If you add a line here, gate it on whatever it claims.**

### The "Progressing" gate is a substring match — know its failure mode

It tests `desc.includes("Build it up")`, which is how the library writes
progressions. Currently accurate at 241 of 311 entries. The nerve glides remain
the thin spot at 5 of 17 — which is why the failure surfaced there first — though
all three glides added in Batch B carry one, so the ratio is improving rather
than drifting.

If a future content batch writes progressions as "Progress by…" or "Work up to…",
those entries silently stop counting and the line disappears from sheets that
deserve it. **If the convention ever loosens, replace the string test with a real
`progression` field** — do not accumulate more substrings.

Hand-typed custom entries were the likeliest source of that drift, and they are
no longer able to cause it: the form takes a separate Progression field and
`assembleDesc()` writes the "Build it up: " sentence itself. A blank field writes
no sentence, so the entry correctly does not count. **Do not ask the user to type
the convention, and do not let a second way of writing it in creep back.**

## Custom exercises

Everything the renderers read comes from `ALL` and `byId` — the built-in `EX`
concatenated with `CUSTOM`. **`EX` is never mutated**, and nothing outside the
custom-exercise section should reference it directly: read `ALL` instead, or a
custom entry silently stops existing for whatever you are writing.

`rebuildLibrary()` is the only thing that reassembles them, and it repopulates
`byId` **in place** because closures all over the file already hold that object.
Call it after any change to the custom library, then `renderFilterChips()` — the
region and type chips and the colour legend are derived from `ALL`, so a custom
entry can be the thing that puts content into a region.

Ids are `custom-{region}-{mob|stab|nrv}-{slug}-{4 random chars}`. The suffix is
what makes two entries typed with the same name distinct, and what makes a
user-typed slug unable to collide with a built-in. **Ids never change on edit** —
saved programs point at them.

Anything arriving from a file goes through `sanitizeCustom()`, which rebuilds the
record field by field rather than trusting it, and through `mergeCustom()`, which
never overwrites an existing id: an incoming clash is kept under a fresh id and
`remapIds()` rewrites the program's references to match. That remap has to happen
**before `migrate()` runs**, or `migrateItems()` drops every reference as unknown.
An incoming entry identical to one already held is recognised as the same entry
and skipped, so re-importing your own export does not breed duplicates.

## The anatomy index

`ANATOMY` is a flat array of 271 structures — muscles, joints, ligaments, fascia
and nerves — each resolving to the library drills that load it. It backs the
Anatomy view, and a separate 3D viewer project consumes the same dataset and
treats this file as its source. **Keep it liftable:** plain data, one top-level
array, no reference to anything that renders it.

```js
{
  id: "shoulder-rotator-cuff",   // stable slug
  name: "Rotator Cuff",
  latin: "mm. supraspinatus, infraspinatus, …",
  region: "shoulder",            // an MVMT region, not an anatomical one
  system: "muscular",            // muscular | skeletal | articular | nervous | fascial
  layer: 2,                      // 1 superficial · 2 deep · 3 skeletal
  action: "…", clinical: "…",
  ex: ["Side-lying External Rotation", …]   // NAMES, not ids

  // both optional, both resolved in resolveAnatomy():
  inherits: "hip-hamstrings",    // show the parent's drills, then this one's own
  noExercises: "A landmark, …"   // why this structure legitimately has none
}
```

Structures take `alsoRegion` on the same terms exercises do, and it is
load-bearing rather than cosmetic: Erector Spinae is filed `lumbar` with
`alsoRegion: ["thoracic","cervical"]`, and that is the only reason the Thoracic
filter still shows an erector entry after the two region-cut ones were retired.
`regionsOf()` is shared with the library; on a structure it carries **none** of
the safety-callout coupling it has on an exercise, because the callouts only ever
read exercises. Adding a structure to `"cervical"` or `"head-jaw"` is therefore
a filing decision, not a clinical one — the opposite of the rule for exercises.

`region` is the MVMT region so a structure's drills and the region filter agree.
That is why psoas is filed under **lumbar** rather than hip, and why the nerves
are filed where you would *look* for them rather than where they end — median
under cervical, tibial under ankle-foot. Anatomically arbitrary, clinically
correct. `layer` is stored and never surfaced; it exists for the 3D viewer.

### `ex` holds names, and an unresolved name is a hard error

There is no build step, so `resolveAnatomy()` does the lookup at load and again
on every library change — it is called from `rebuildLibrary()`, because a custom
exercise can satisfy a name and deleting one can take a name away again.

Exact match first; then a case-and-whitespace-insensitive fallback. **The
fallback still reports**, because a match that needed it means the map's wording
has drifted from the library's. A name that resolves neither way is reported to
the console *and* to a banner at the top of the Anatomy view, and is flagged on
the structure itself.

This only works while names are unique, which is why `resolveAnatomy()` also
asserts that — see **`name` is a resolution key** above. A duplicate is the worse
failure of the two: an unresolved name shows up as a short list, a duplicated one
shows up as a plausible wrong drill.

> **Never let a name fail quietly.** A structure that silently loses a drill is
> invisible — a clinician sees a short list and has no reason to distrust it.
> The structure keeps the drills that did resolve; it is never emptied.

### `inherits` and `noExercises`

Both are optional, and both are now in use.

- `inherits` — the six anterior forearm flexors (FCR, FCU, palmaris longus, FDS,
  FDP, FPL) inherit from `elbow-wrist-flexors` ("Wrist Flexors"). Each child
  states only what is specific to it; the four shared flexor drills are written
  once, on the parent.
- `noExercises` — one entry, `nerve-facial`. Facial retraining is specialist
  work, so the map says so rather than inventing a drill for it.

`inherits` is now two things at once, and the second is why it must not be
used for loose association: **it is the sole input to the group/detail toggle.**
A structure with a parent is a *part*; anything else is a group or a standalone.
Nothing else distinguishes them, and no field was added to.

> **Show named parts** (Anatomy view, default **off**) hides every structure with
> an `inherits`. Erector Spinae reads as one entry; turn it on and the nine
> named parts appear nested beneath it.

Three rules that toggle has to keep:

- **Search ignores it.** `anatPasses()` returns on the query *before* consulting
  the toggle. A structure you typed the name of and still could not see would be
  a bug, not a filter working.
- **It never persists.** Module state, same reasoning as the patient-view toggle:
  the default has to be what a reload lands in.
- **The hierarchy stays navigable with it off.** A parent lists its children as
  links under *Breaks down into*, a child links back under *Part of* — so a
  hidden part is always one click away, and that is what makes defaulting to off
  safe.

**Reparenting an entry removes it from the default browse list.** That is the
toggle working, not a bug, but it is a real cost and it lands on exactly the
entries you reach for most: Piriformis, Upper Trapezius and Rectus Femoris all
became parts in Batches E–G and none of them appears in the default list any
more. Three things make that acceptable, and a fourth would not survive losing
any of them — search finds a part regardless of the toggle, the parent lists it
under *Breaks down into*, and the parent is itself the thing you usually wanted.
**Weigh that before making a well-known entry a child of something.** The test is
whether someone looking for it would accept the parent as the answer.

Nesting is by `inherits` within a region group. A part whose parent is filtered
out, or files under a different region, renders flat and carries a **Part** tag
instead of an indent — better a row in a slightly odd place than a row silently
missing. The same instinct guards the walk itself: a structure an `inherits`
loop keeps out of the tree is appended flat rather than dropped, since
`resolveAnatomy()` already reports the loop.

`inherits: "<parent-id>"` makes a split structure show **the parent's drills
first, then only what it adds**, deduplicated. Splitting hamstrings into three or
erector spinae into nine otherwise means restating the same shared list on every
part and re-verifying it each time. Resolution is recursive and memoised, so a
child may be declared above its parent and a chain costs one walk.

`noExercises: "<reason>"` is the reason a structure legitimately has none —
landmarks, most bones, several nerves. The Anatomy detail panel prints the reason
under **No drills load this** instead of an empty section.

Pair it with an explicit `ex: []`. The loader tolerates the key being absent, but
the 3D viewer consumes this array as data and should not have to guard for it.

A structure carrying `noExercises` is making a clinical statement, not filling a
hole — `nerve-facial` says *refer rather than prescribe*. **The map should be
willing to tell you not to treat something**; that is a better answer than four
loosely related drills.

**An empty `ex: []` is legal only with a reason.** Without one there is no way to
tell a structure that has no homecare from one whose every name failed, and that
ambiguity is precisely the silent failure the index refuses to have. So
`resolveAnatomy()` reports, to the console and to the Anatomy banner and on the
structure itself:

- an empty drill list with no `noExercises`
- a `noExercises` on a structure that does resolve drills
- an `inherits` naming a structure that isn't in the map, or forming a loop

In each case the structure still shows whatever it legitimately resolved — the
defect is surfaced, not compensated for by hiding it.

### `clinical` is practitioner-only and must never print

`action` is plain enough to read aloud to a patient. `clinical` names
compensation patterns and things people get wrong — useful to CYU, unhelpful or
alarming to a patient reading over a shoulder. They are separate fields so
hiding one is a toggle, not a rewrite. **Do not concatenate them.**

Three things keep `clinical` off paper, and all three have to stay true:

1. `#anatView` carries `no-print`, whose rule is `display:none!important` —
   which beats the inline `display` the views are toggled with.
2. Nothing copies anatomy text into a program. Add-to-handout pushes the same
   id-only item the library's own checkbox does, so `clinical` cannot reach a
   saved program, an export or a handout by that route.
3. `renderPatient()` never reads `ANATOMY`.

The Patient view toggle hides `clinical` and nothing else. It is **module state,
never storage** — the safe state is practitioner, so that has to be the state a
reload lands in. Do not make it sticky.

## Storage

```
homecare:programs   → array of program objects
homecare:custom     → user-authored exercises, merged over the built-in library
homecare:settings   → { practitioner, lastProgramId }
rtt:program         → legacy v1 key, read once on first load, never written
```

`migrate()` accepts v1 or v2 and is the only place that knows `LEGACY_ID_MAP`.
Unknown exercise ids are dropped and reported to the user, never silently.

Autosave debounces 800ms. Any code path that replaces the open program must call
`flushSave()` first, or the pending write lands on the wrong program and the
previous one's last edit is lost.
