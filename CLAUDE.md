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

## `alsoRegion` is coupled to a safety callout — read before cross-tagging

`alsoRegion` exists for **discoverability**: it cross-lists an exercise under one
or more further region filters without changing its `id`, so no migration is
needed. It takes a string or an array — the ten scapular drills filed under
`region: "thoracic"` carry `alsoRegion: "shoulder"`, and single leg balance is
`["knee","ankle-foot"]` on top of its hip primary.

**But it is not purely cosmetic.** The patient handout renders the neck safety
callout whenever a phase contains any exercise that reports `"cervical"` from
`regionsOf()` — which reads `alsoRegion` as well as `region`. So:

> Cross-tagging an exercise into `"cervical"` silently attaches the neck safety
> callout to every handout that contains it.

That widening is deliberate — for a safety callout, firing on any cervical
involvement is the correct failure direction. The consequence is that
`alsoRegion: "cervical"` is a **clinical** decision, not a filing one.

**Never cross-tag into `"cervical"` for filter convenience alone.** If an
exercise should appear under the Cervical filter but genuinely does not warrant
the neck warnings, the callout logic needs to change first — do not tag and hope.

Cross-tagging into any other region carries no such side-effect today. If a
future region gains its own forced callout, this note needs updating alongside it.

The manual-entry form exposes `alsoRegion` as a dropdown, so this decision is now
made by whoever is typing rather than by whoever is editing the library. That is
why the field carries an inline note naming the consequence. **Keep that note
truthful if the callout rules change.**

## Exercise schema

```js
{
  id: "hip-mob-9090-switch",   // {region}-{mob|stab|nrv}-{name-slug}, stable
  region: "hip",               // cervical|shoulder|elbow-wrist|thoracic|lumbar|hip|knee|ankle-foot
  alsoRegion: "shoulder",      // optional, string or array — see the coupling warning above
  type: "mobility",            // mobility | stability | nerve
  level: 1,                    // 1 Foundational | 2 Intermediate | 3 Advanced
  name: "90/90 Hip Switch",
  desc: "...",                 // clinical copy handed to patients — do not reword
  targets: "...",              // likewise
  rx: "2 x 8 each side, slow"  // default prescription; every entry must have one
}
```

`desc` and `targets` are written to be read by patients. Do not rewrite,
summarise or "improve" them without being asked.

User-authored entries use this same shape and are indistinguishable to every
renderer. They carry four fields on top of it — `custom: true`, the raw
`description` and `progression` the form was filled in with, and `createdAt` /
`updatedAt`. `desc` is stored **assembled** from the first two, so nothing
downstream has to know the difference. See the custom-exercise section below.

### "Tennis elbow" is a deliberate, approved exception

The tool was rebranded away from its origin as a return-to-tennis builder, and
one acceptance check was written as *"no occurrence of 'tennis' in any UI-facing
string"*. Three occurrences survive on purpose, all in elbow-wrist `targets`:

- "the group involved in tennis elbow"
- "the reliable starting point for a painful tennis elbow"
- "Tendon loading for tennis elbow"

**Do not "fix" these.** Tennis elbow is lateral epicondylitis — the condition
name patients actually use. It is clinical vocabulary, not sport framing. The
acceptance check was written too literally; the rule it was protecting is that
nothing frames the tool around a sport.

If you grep for `tennis` and get exactly these three hits in `targets`, that is
the correct state. More than three, or any hit outside `targets`, is not.

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
   The region colours are a scanning aid across a 281-entry list, so a ninth one
   has to stay distinguishable from the other eight **and** from `--accent` —
   see the theme section.
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

Both are non-dismissible and render automatically. Where a phase triggers both,
**neck prints first, nerve second** — the order comes from the render sequence in
`renderPatient()`, so keep `${cervicalCallout}` above `${nerveCallout}`.

| Trigger | Callout |
|---|---|
| any exercise with `"cervical"` in `regionsOf()` | About the neck exercises |
| any exercise with `type: "nerve"` | About the nerve glides |

A cervical nerve glide raises **both**, which is correct: it is cervical by
region and a nerve glide by type, so both sets of rules apply.

Both render as `<div class="callout safety">`. The therapist's note and the
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
progressions. Currently accurate at 216 of 281 entries (and only 2 of the 14 nerve
glides, which is why the failure surfaced there first).

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
