# Attribution for the files in this folder

The `.glb` models here are the runtime export of
[mvmt-anatomy](https://github.com/calvinyu94-debug/mvmt-anatomy), which holds
the pipeline, the verification renders and the full attribution chain. In
summary:

- The musculoskeletal geometry — `<region>.glb`, `<region>-insertions.glb`
  and `overview.glb` — derives from **Z-Anatomy** (https://github.com/Z-Anatomy), licensed
  CC BY-SA 4.0, created by Gauthier Kervyn and contributors. Z-Anatomy is
  itself derived from **BodyParts3D** (Database Center for Life Science,
  Japan), licensed CC BY-SA 2.1 Japan. Structure naming follows Terminologia
  Anatomica (TA2, 2019). The components of Z-Anatomy that carry non-commercial
  or unstated licences are excluded from these exports; `EXCLUSIONS.md` in
  mvmt-anatomy is the authoritative list.
- `nerves.glb` is **original work** from mvmt-anatomy: the ten peripheral
  nerves Z-Anatomy does not have, authored as schematic tubes from a written
  specification and checked against the real bones. They are indicative paths,
  not imaging-derived anatomy, and the viewer says so wherever they are shown.
  They are released CC BY-SA 4.0 by this project; neither upstream is their
  source and neither should be cited as it.
- `regions.json` is a trimmed record of what was exported — file names, sizes
  and triangle counts — produced from the pipeline's manifest.
  `structure-meshes.json` is the structure-to-mesh join, trimmed to ids and
  node names, and `landmarks.json` holds the resolved coordinates of the 42
  palpable landmarks. Both are records of the same pipeline.
- `overview.glb` is the whole musculoskeletal model decimated to roughly a
  twentieth — skeletal, muscular and articular meshes only, with no context
  layer and no insertion patches, which stay per-region. Same source and same
  licence as the region files: it is the same geometry, coarser.
- The **fascial lines** in `structure-meshes.json` are a second index over
  structures that already exist — no geometry is authored for them. The twelve
  myofascial continuities, and the order of the tracks within each, are derived
  from Thomas Myers, *Anatomy Trains*, 3rd edition, cross-referenced against
  this map. They are an argued model rather than dissected structures, and the
  viewer says so wherever one is drawn.

Everything in this folder is distributed under **CC BY-SA 4.0**.
