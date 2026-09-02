# mvmt-program

Printable home-exercise program builder for clinical use.

**Live:** https://calvinyu94-debug.github.io/mvmt-program

## How it works

A single self-contained `index.html` — no backend, no build step, no accounts.
Open the live URL in any browser and it runs.

- **Build** view is the practitioner working view.
- **Patient copy** is the clean handout, laid out for printing.
- **Print / Save as PDF** produces the patient-facing sheet.
- **Anatomy** is the structure index: what each muscle, joint and nerve does
  and which drills load it.
- **3D** is the anatomy viewer. It loads nothing until it is opened — the
  libraries and the region models arrive on demand, one region at a time, so
  the rest of the app stays exactly as fast as it is.

## Where your data lives

Programs are saved in your browser's own storage (`localStorage`), on the device
you're using. Nothing is uploaded anywhere and nothing is stored in this
repository — the site is public, so treat it that way.

Because the data is per-device and per-browser:

- Clearing browsing data will erase saved programs.
- A program saved on the iPad will not appear on the desktop.

Use **Save file (.json)** to keep a portable backup, and **Open file** to load it
back or move it to another device.

## Editing

Everything lives in `index.html`, except the 3D models, which are the `.glb`
files in `assets/`. Those derive from Z-Anatomy and are CC BY-SA 4.0 — see
[`assets/ATTRIBUTION.md`](assets/ATTRIBUTION.md). Changes pushed to `main` go
live automatically via GitHub Pages.
