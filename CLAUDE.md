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

The only other place literals appear is inside `@media print`, in two further
`:root` blocks: the ink-saver swap, which re-points the same tokens at flat
white, and `:root.print-colour`, which lifts the two print-wash tokens — see
the print colour section below. Print is a token swap, deliberately, rather
than a pile of overrides scattered through the rules. The one-line check above
has to strip all three blocks, and the third carries a class on its selector,
so match `:root…{` rather than `:root{`.

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

### Print carries meaning through ink, never through fill alone

Same standing as the two rules above, and it was shipping broken for as long
as the Build printout has existed.

**Backgrounds are dropped by default in print.** Save as PDF and Print both
discard background fills unless the user ticks Background graphics, which is
off by default. Any element that relies on a background fill for legibility
prints as invisible: the level tag was white text on an ink fill, and printed
as white text on nothing. Nobody noticed, because an invisible tag leaves no
trace to notice.

> Print styling must carry meaning through ink — borders, weight, position —
> never through fill alone. Colour mode is additive and must never be the thing
> that makes something readable.

So a print rule for a filled element puts ink text on `--panel` with a real
border, and it does so **in both modes**. A tag that is legible only with the
colour toggle on is the same bug in a smaller form, because the toggle is off
by default and off on every machine that has not chosen otherwise.

The test is not "does it look right in print preview" — preview honours the
Background graphics setting of the moment. It is: **with every background
fill removed, does every piece of text still contrast with white, and does
every box still have an edge?** The audit that found the tag walked every
rendered element on each printable view with the print block applied, and
flagged text whose colour falls under 3:1 against white and boxes with a fill
and no border. Re-run it when adding anything to a print path that carries
a fill: filled tags, selected chips, pressed states, checkboxes drawn with
`accent-color`.

The audit's full list is now fixed, and the fix shows what the rule looks like
in practice: every tag and the selected filter chip print as ink on
`--panel` with a 1px rule. The screen hierarchy — ink fill, accent fill, light
fill — survives as **rule weight**: `--ink` for the level tag, the Custom tag
and the selected chip, `--ink-3` for the plain type and region tags. The
selected chip also takes `font-weight:700`, so its state does not rest on one
hairline. The chip was the one that mattered: a printed filtered library was
showing every unselected chip and hiding the selected one — the inverse of the
actual state, which is worse than missing.

Two things it is easy to reach for that do not satisfy this:

- `print-color-adjust: exact` on the element. That is colour mode's tool, and
  it fails the rule above — the element is readable only because a fill
  survived.
- A lighter text colour that "reads on both". Text that has to be readable on
  white and on ink at once is readable on neither.

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

### Print colour mode is additive, and off must stay byte-identical

The print block holds two `:root` blocks, not one. The first is the ink-saver
swap that has always been there — every tinted token to flat white — and it is
the default output: the one that has been reviewed and is in clinical use.
**Nothing in it may change on account of colour mode**, and nothing about
colour mode may reach the page unless `<html>` carries `print-colour`. The
check is that a sheet printed with the toggle off is byte-identical to one
printed from before the toggle existed. It was verified with headless Chrome
against `main` when the mode landed, and it should be re-verified the same
way after any edit to the print block.

The second, `:root.print-colour`, lifts exactly two tokens — `--print-wash`
and `--print-wash-2` — which the ink-saver block declares white. They are
their own tokens rather than a re-pointing of `--accent-wash` and
`--panel-2`, because those two also ground selected library rows, group
headers and the notes block, none of which comes back in colour mode.
**Colour mode is not a screenshot of the screen.** What keeps its colour is
the set of elements whose colour carries meaning, and nothing else:

| Element | Wash |
|---|---|
| `.callout.safety` — the three forced callouts | `--print-wash` (accent) |
| `.callout` — therapist's note, How to run this program | `--print-wash-2` |
| `.pex .prx` — the dose pill | `--print-wash` |

Page, panel and sheet backgrounds stay white in both modes. Daily
modifications is not in the table and should not be added: commit e8d3f94
deliberately demoted it from a callout to a plain numbered Part, and it
carries no wash on screen. Giving it one in print would reverse that decision
by the back door.

Two things make this a class rather than a token swap, and both have to hold:

- **Save as PDF and Print both discard backgrounds** unless the user finds the
  Background graphics box, which is off by default. Removing the white swap
  would not have been enough: every element in the table carries
  `print-color-adjust: exact` and the `-webkit-` form, and that is what makes
  the wash survive. An element added to the table needs both lines.
- **The borders are not colour mode's job.** The ink-saver block already
  restores the callout rule and the pill's accent border, and those rules
  still apply. Colour is additive, not a replacement for structure.

Contrast was measured on the washes, not on white, for the heading (`--ink-3`)
and the body ink — and again with text and ground both converted to grey,
because a mono clinic printer renders the wash as a light grey rather than
dropping it:

| | `--print-wash` | `--print-wash-2` |
|---|---|---|
| `--ink-3` heading | 4.50:1 (4.53 in grey) | 4.62:1 (4.65 in grey) |
| `--ink` body and pill | 13.99:1 | 14.36:1 |

The heading on the safety wash is the binding case at exactly 4.50 — the same
floor `--ink-3` was set against. If a printer ever renders the wash darker
than that, **lighten `--print-wash` in the `:root.print-colour` block** —
that is what a print-only token is for. Do not lighten `--ink-3`, which
cannot move (see above), and do not retreat from the mode.

The toggle is a checkbox beside Print, not a third button, so printing stays
one click. It is stored as `printColour` in `homecare:settings` and persists
deliberately: it is a practitioner preference, not a safety control, so the
reasoning that keeps the Anatomy patient-view toggle out of storage does not
apply. It shows and hides with the print button, so the Anatomy view carries
neither. The class is set on `<html>` whatever view is open, so it governs
the Build printout on the same terms as the handout — Build simply has nothing
in the table today.

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

`ANATOMY` is a flat array of 335 structures — muscles, joints, ligaments, fascia
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
  system: "muscular",            // muscular | skeletal | articular | nervous
                                 // | fascial | landmark
  layer: 2,                      // 1 superficial · 2 deep · 3 skeletal
  action: "…", clinical: "…",
  ex: ["Side-lying External Rotation", …]   // NAMES, not ids

  // all optional, all resolved in resolveAnatomy():
  inherits: "hip-hamstrings",    // show the parent's drills, then this one's own
  noExercises: "Specialist work" // why this structure legitimately has none
  attachments: ["Masseter", …]   // landmarks only — STRUCTURE names, not exercise
}
```

### Landmarks answer a different question

A landmark is not a tissue. It is **where you put your hands**, and what you point
at when explaining something — so it carries `attachments` where everything else
carries `ex`, and the detail panel prints **What attaches here** instead of
**Drills that load this**.

`system: "landmark"` is the whole signal. It needs no `noExercises`: the type
already says why there are no drills, and 42 entries repeating that sentence
would be noise. In exchange it is held to the mirror-image requirement — a
landmark with **no attachments** is a defect for exactly the reason an empty
`ex` without a reason is: an empty list is otherwise indistinguishable from every
name having failed. `resolveAnatomy()` also reports a landmark carrying drills, a
landmark carrying `inherits` (landmarks are never parts, so they show whatever
the parts toggle is set to), and a non-landmark carrying `attachments`.

**`attachments` resolve by structure name, and that makes structure names a
resolution key** — the same status exercise names have had since the Wall Ball
Circles clash, and asserted the same way: grouped on the same
case-and-whitespace-insensitive key, reported to the console, to the banner and
on the structure, and re-run on every library change rather than once at boot.
Two structures under one name means only the first is reachable by any landmark
that meant the other.

Each resolved attachment renders as a link to that structure, so a landmark is a
way into the map rather than a leaf of it.

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
homecare:settings   → { practitioner, lastProgramId, printColour }
rtt:program         → legacy v1 key, read once on first load, never written
```

`migrate()` accepts v1 or v2 and is the only place that knows `LEGACY_ID_MAP`.
Unknown exercise ids are dropped and reported to the user, never silently.

Autosave debounces 800ms. Any code path that replaces the open program must call
`flushSave()` first, or the pending write lands on the wrong program and the
previous one's last edit is lost.

## The 3D viewer

The first thing that lives outside `index.html`. The models are in `assets/`:
nineteen `.glb` files — nine regions, nine insertion layers, `nerves.glb` —
plus `regions.json`, all exported by the mvmt-anatomy pipeline and served
same-origin from Pages so there is no CORS and the browser caches them like
anything else. `assets/ATTRIBUTION.md` carries the licence chain: CC BY-SA
throughout, and `nerves.glb` is original schematic work rather than a
derivative. Keep it accurate if a file is replaced.

### Nothing 3D loads until the 3D view is opened

The app runs on clinic wifi, and Programs, handouts and the Anatomy index must
stay exactly as fast as they are. So boot fetches no three.js, no decoder and
no model — **the check is the network tab on a fresh load**, not a reading of
the code. Everything the viewer needs is imported inside `v3Setup()`, which
runs on the first `setView("3d")` and never before. The import map in `<head>`
is not a fetch: it only says where `three` and `three/addons/` resolve once
something imports them, and it is the whole of the "build step". Script tags
and an import map, no bundler — that stays true.

Regions load one at a time and never all nine. `v3Load()` disposes the
previous region's geometries and GLTF-built materials before the next lands
(`v3DisposeRegion()`), and a load superseded while in flight is disposed
rather than shown (`V3.token`). Verify with
`V3.renderer.info.memory.geometries` across several switches: it should equal
the current region's mesh count after a render, and never climb.

### The CDN split, and why the decoder is preloaded

cdnjs carries only the three.js core module, nothing under `examples/`. The
loaders, the controls and the Draco decoder therefore come from jsDelivr,
pinned to the same release — `V3_VERSION` in the script and the number in the
import map are the same release and **move together**. The decoder is
`preload()`ed at setup so a bad decoder URL fails at setup with the viewer's
own message, instead of at the first compressed mesh, where it looks like a
broken model rather than a broken URL.

### Failure leaves the rest of the app alone

Every 3D path is caught and reported on the canvas overlay, in plain words,
with a retry. Nothing 3D may throw into the rest of the page: a model that
will not load must never take a handout with it. The view carries `no-print`
like the Anatomy view, hides the print controls the same way, and is exempt
from the needs-a-program rule in `setView()`.

### The viewer's colour is still theme colour

The `--v3-*` tokens in `:root` are the viewer's whole palette — ground, key
and fill light, one colour per tissue, one for context. The viewer reads them
by name through `getComputedStyle` and holds no literal of its own, so the
one-line colour check still comes back empty and a theme swap reaches the
model. Meshes are coloured **by tissue**, from the node's `system` extra with
the exporter's material name breaking the ties a hand cares about (tendon,
bursa, ligament) — never by region. The region tokens are a scanning aid
across a list, and mean nothing wrapped round a femur. The exported materials
themselves are Z-Anatomy's functional-group atlas, 37 names and nearly all
plain white, so they are replaced on load and disposed.

### The glTF extras are the contract with mvmt-anatomy

Every node carries `sourceName`, `region`, `side`, `context` and `system` in
its extras; the nerve nodes carry `authored: true` and `source: "schematic"`
and no `sourceName`. Context meshes (`context: true`) are the neighbouring
regions, drawn dim with depthWrite off so they never occlude the region
itself, and are not selectable. Coordinates are baked to world space with
identity transforms, glTF Y-up, **+Z anterior**, `.l` at +X.

### `.gitignore` negations are exactly three

`*.json` stays blanket, and a patient export dropped in the repo folder must
still be ignored — in the root and inside `assets/`, and that is worth
re-checking with `git check-ignore` after any edit to the file. The three
negations are the three runtime data files and nothing broader. `.glb` was
never ignored; `.gitattributes` marks it binary so it can never be
line-ending normalised.

Two smaller things the viewer keeps: it respects `prefers-reduced-motion` by
turning orbit damping off, and it caps the device pixel ratio at 2.

### Selection resolves through the join, and a mesh may belong to nobody

`assets/structure-meshes.json` is the structure-to-mesh join from
mvmt-anatomy, trimmed to `id`, `kind`, `meshes` and `insertions` — names,
regions and triangle counts are not repeated because `ANATOMY` has them.
`kind` is one of `mapped`, `absent`, `unresolved`, `anchor` (a landmark),
`authored` (a schematic nerve) or `text-only`; only `mapped` and `authored`
carry geometry. It was transcribed by hand and checked three ways: every
mesh and insertion name exists in the export manifest, every id matches
`ANATOMY` in both directions, and per-structure triangle sums reproduce the
join's own figures. **Do not edit it by hand without re-running that check.**

A click raycasts against the region's own meshes only — context is never in
the list — and reads the hit's `sourceName` back through the join. A mesh
can be claimed by several structures (Piriformis is its own entry and part of
the deep rotators), so the most specific claim wins, the one with the fewest
meshes, and the rest are offered under *Also part of*. Selecting a structure
highlights every mesh it claims, both sides. Roughly a third of the exported
objects are claimed by nothing; a click landing on one says *Not in the
index* and names the mesh. That is ordinary, not an error, and it must never
be turned into silence.

Nerve nodes carry no `sourceName`, and GLTFLoader sanitises node names on the
way in — `nerve-femoral.l` arrives as `nerve-femorall`. They are matched by
sanitising the join's names the same way, never by parsing the loaded name
back. If the loader's rule changes, that one function changes with it.

### The depth slider peels; it never deletes

Four stops: Skin, Superficial, Deep, Bone. A mesh's depth is the shallowest
`layer` of any structure that claims it — 1 superficial, 2 deep, 3 skeletal —
and an unclaimed mesh falls back to its `system`: skeletal and insertions
are 3, articular 2, everything else 1. Skin is the model as exported. At each
later stop the layers above are hidden, the layer at it is drawn as normal,
and the layers beneath stay visible faded, so a deep muscle is still seen
against the bone it works on. The selected structure is always shown and
always full whatever the slider says: it is what was asked for. Context,
landmarks and nerves stand outside the depth system.

### Landmarks: coordinates from one file, everything else from the other

`landmarks.json` is the only source of a marker's position and is read for
nothing else — its region keys (`head-neck`, `pelvis`, `foot` …) are not the
app's, so which region a landmark belongs to, and what it is called, come
from `ANATOMY`. The file is Blender Z-up in metres; the export turned +Z into
+Y and −Y into +Z, so a point `(x, y, z)` lands at `(x, z, −y)`. Paired
landmarks have `l` and `r`, midline ones `m`; every side gets a marker.
Selecting a marker shows *What attaches here* from `anatAttach`, and each
attachment goes through `v3Show()`, which loads the region that holds the
structure if this one does not.

### The schematic treatment is the debt to mvmt-anatomy, and it is not optional

The authored nerves in `nerves.glb` were written from a specification; the
muscles and bones come from cadaveric and imaging data. Drawn the same way
they would look equally authoritative, and they are not. mvmt-anatomy's
`CLAUDE.md`, `README.md` and `ATTRIBUTION.md` all record the viewer-side
treatment as owed. It is paid here, in two parts that both have to stay:

- a **distinct material** — the nerves are drawn unlit and flat, no shading
  and no roughness, so they read as an overlay on the model rather than a
  part of it (`V3_LOOKS.nerve`, `flat:true`);
- the label **"Schematic — indicative path only"** (`V3_SCHEMATIC`), shown
  over the model whenever the nerve layer is on or a nerve is selected, and
  again in the selection panel beside the nerve's name. It has no close
  control. It is not a tooltip. **Do not soften it for visual tidiness** —
  that is exactly the failure the requirement names.

Only the ten authored nerves have geometry; the other nineteen are text-only
by design and the viewer shows nothing for them. Which nerves appear in a
region is a heuristic: a nerve is shown if `ANATOMY` files it under the
region, or if at least a third of its sampled vertices lie inside the boxes of
the region's own meshes. Bounds alone were too coarse — the arms hang through
the hip's box, and the rotatores are one mesh from neck to sacrum.

The debt is closed in mvmt-anatomy's three files only once the integration
PR lands, with a pointer to where it was paid — not before.

### The viewer is a way into the handout, and the index and the viewer say one thing

Tap a muscle, read what it does, add the drills, print the handout: that
chain is the product, and three things keep it honest.

**One detail body.** `anatDetailBody(s)` renders what a structure does, why
it matters, and the drills that load it, and both the Anatomy index and the
3D panel call it. The caller supplies the head and handles the clicks; the
body never differs between the two, so a drill list or a clinical note can
never be right in one place and stale in the other. Adding a drill from the
3D panel goes through `anatAddToPhase()`, the same function the index uses,
which pushes the same `{exerciseId, rx, freq, note}` item the library's own
checkbox does. No anatomy text travels with it.

**One patient toggle.** `anatPatient` is the index's variable and the 3D
panel reads the same one; the `Patient view` button on the 3D side flips it
and the index's button shows the new state the next time that view renders.
Module state, never stored — a reload lands in practitioner mode in both
views, for the reason given under the Anatomy index.

**The three print guards hold from here too.** `#threeView` carries
`no-print`; the 3D panel copies nothing into a program; `renderPatient()`
still never reads `ANATOMY`. The check that was run: every rendered element
carrying a `clinical` sentence sits under a `.no-print` ancestor (the only
other carrier is the script source itself), the print block's `.no-print`
rule is `display:none!important`, and the patient copy's HTML contains no
clinical text after a drill has been added from 3D.

### View in 3D is offered from a generated set, not a fetch

The Anatomy index shows **View in 3D** only for structures the join gives
geometry — `mapped`, `authored` or `anchor` — and hides it for `text-only`,
`absent` and `unresolved`. It cannot read `structure-meshes.json` to decide,
because the index must fetch nothing 3D; so `V3_HAS_GEOMETRY` is a generated
`Set` of ids in the script, with the regenerating one-liner in the comment
above it. At 3D setup the set is checked against the file and any drift is
reported to the console and beside the layer toggles. **Regenerate it
whenever the join changes**; never hand-edit it.

`v3Show(id)` is the one entry point from the index, from an attachment link
and from *Also part of*: it loads the region that holds the structure if the
current one does not, turns on whichever layer the structure lives in
(landmarks, nerves, or insertions for a structure that has patches and no
belly), selects it and frames it. Framing is the viewer's single camera
animation, half a second eased, and it is a jump under
`prefers-reduced-motion`. A request made before the viewer has finished
setting up is kept in `V3.pendingShow` and replayed, never dropped.
