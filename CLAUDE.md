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

Every `id` must start with its own `region` key. Ids are referenced by saved
programs, so changing one costs a `LEGACY_ID_MAP` entry — get them right before
content ships, not after.

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

## Storage

```
homecare:programs   → array of program objects
homecare:custom     → user-authored exercises (not built yet)
homecare:settings   → { practitioner, lastProgramId }
rtt:program         → legacy v1 key, read once on first load, never written
```

`migrate()` accepts v1 or v2 and is the only place that knows `LEGACY_ID_MAP`.
Unknown exercise ids are dropped and reported to the user, never silently.

Autosave debounces 800ms. Any code path that replaces the open program must call
`flushSave()` first, or the pending write lands on the wrong program and the
previous one's last edit is lost.
