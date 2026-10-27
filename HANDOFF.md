# Hand-off notes

A talk about the lessons learned converting large microscopy volumes to
OME-Zarr 0.6 for neuroglancer, and about using a public Globus guest
collection to share data instead of GIN. It was drafted on 2026-10-27 in
the `amf-convert` repo (`~/NIU-dev/amf-convert`) and moved here for further
development.

## Layout

- Everything is in `index.qmd`. The code blocks are static and no cells
  are executed, so the deck builds without Python or the example data.
- The deck is in seven parts:
  1. Deciding what the output looks like: metadata, chunks, shards, codec,
     pyramid.
  2. Getting the bytes across: the naive path, band write order, building
     the pyramid on the way past, the exact mean.
  3. Memory: three OOMs and the RAM formula.
  4. Speed: the read-share diagnostic, zarr-python → tensorstore, storage.
  5. Running at scale on Slurm.
  6. Looking at it in neuroglancer: serving, quirks, state files.
  7. Globus as a public file host: comparison with GIN, setup, pooch for
     CI, cautions.
- Section title slides use `background-color="#03A062"` (niu-green from
  `_brand.yml`).

## Building and checking

    quarto render index.qmd     # -> build/index.html

`_quarto.yml` sets `scrollable: true`, so a slide that's too long scrolls
instead of being clipped. That means overflow is easy to miss in the
editor. Dense slides carry `{.smaller}`. To check every slide, screenshot
it with headless Chrome. Slides are addressed as `#/<section>/<slide>`, and
the first four slides come before any section:

    google-chrome --headless=new --hide-scrollbars --window-size=1280,800 \
      --virtual-time-budget=4000 --screenshot=s.png \
      "file://$PWD/build/index.html#/4/1"

## Left over from the template

- `footer` is the talk title. Change it to "Location | Event | Date" once
  the event is known. `date: today` comes from the template.
- `slides/extra_slide.qmd`, `references.bib` (the blackcap entry) and
  `README.md` are template examples. The deck doesn't use them.

## Where the numbers come from

All figures are measurements from the `amf-convert` work. Keep the setup
next to each throughput number when editing, because figures from
different disks aren't comparable.

| Claim | Setup / source |
|---|---|
| 2123×8879×6030 uint16, 2.708×2.708×5 µm, 227 GB per channel | Imspector (LaVision Blaze) OME-TIFF example |
| 514,930 files vs 1,283 shards | computed from `amf_convert.pyramid.level_factors`, 8 levels, 64³ / 512³ |
| zstd-6 1.68 at 128 MB/s/core; blosc-zstd3-shuffle 1.80 at 887 MB/s/core | real 64³ chunks from that scan |
| naive ngff-zarr 0.10 GB/s, ~38 min | desktop, NTFS (ntfs-3g) SSD |
| row vs cube reads 380 vs 34 MB/s; 8 parallel readers 290 vs 400 MB/s | same desktop NTFS disk |
| s0+s1 21.7 min at 0.17 GB/s; s2–s7 ~1.2 min; 95 GB in 1,292 files | full run of one channel, same disk in and out |
| ~50% of voxels off by 1; mean at 2.29 vs 0.44 GB/s | streamed pyramid vs ngff-zarr; integer vs float mean |
| 100 GiB MaxRSS in ~6 min (dask pickling the memmap) | cluster Slurm job; `sacct` plus `faulthandler` dumps |
| 41 GB RSS from queued parent regions | desktop, before the bounded pool |
| RAM ≈ 3 GB + (2.25·workers + 1)·band | measured 5.6 bands at 2 workers, 10.1 at 4 |
| zarr-python 0.04–0.08 GB/s → tensorstore 0.36 GB/s | cluster (Ceph), 2-channel OME-TIFF |
| reads 1.56 GB/s, reader busy 32% | cluster, RC_Ai14_SOM_02 BrainSaw, 593 GB, 2 ch |
| BrainSaw: 7,650 files, 3 ch, 293 GB | MS_cx_left acquisition |

The converter code these lessons describe is in `amf-convert`:
`scripts/ome_tiff_to_zarr.py`, `scripts/brainsaw_to_zarr.py`,
`scripts/ngl_state.py` (standalone), and `amf_convert/ome_zarr.py`.

## Verified vs not

- **Verified:**
  - `pooch.retrieve` downloads from the public guest collection with no
    login.
  - GIN (tested on `neuroinformatics/movement-test-data`) ignores Range
    headers and sends no CORS headers.
  - Neuroglancer streams the sharded stores from Globus.
- **From docs and university guides, not clicked through by us:** the
  guest-collection setup steps (Share → Add Guest Collection,
  public (anonymous) read, Get Link). Check them against our institution's
  Globus Connect Server before presenting.
  [Globus HTTPS access docs](https://docs.globus.org/globus-connect-server/v5/https-access-collections/)
  say the HTTPS server doesn't list directories.
- **Unverified:** Fiji's support for OME-Zarr 0.6. Neuroglancer's 0.6
  support was only in `master` as of 2026-10-21.
- **Left out on purpose:** Globus refuses suffix ranges (`bytes=-N`,
  HTTP 416), so tensorstore and zarr-python can't read sharded stores
  from it. Neuroglancer is unaffected, because it sends a HEAD request and
  then an explicit range. It's out of the talk because CI fetches whole
  files and only neuroglancer reads the OME-Zarr. Bring it back if Python
  streaming ever becomes a use case.

## Rules kept so far

- Real Globus hostnames, Ceph paths and node names are placeholders
  (`https://<collection>.data.globus.org/...`).
- Never quote BrainSaw `recipe_*.yml` files. One holds a Slack webhook
  token, which is the example on the "Before you make it public" slide.
- Serving is Globus only. Don't mention local HTTP servers (`npx
  http-server`, `serve_zarr.py`).
- BigDataViewer/HDF5 is out of scope. The talk covers OME-Zarr only.

## Ideas not done yet

- Neuroglancer screenshots (into `img/`) and a live demo link.
- Draw the ASCII band diagram as a proper figure.
- Replace the pooch example with a real registry once test data is hosted.
