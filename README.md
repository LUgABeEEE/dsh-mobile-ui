# dsh-mobile-ui

**Measure the DSH mobile surface, then fix what the measurement finds.**

DeepSeek Harness is desktop-first. On a phone — the dedicated `dsh-mobile` surface or the stock web UI at a phone width — the layout mostly works, but third-party plugin panels are where it falls apart: buttons a thumb cannot hit, content hidden behind a floating widget, rows crowded against a navigation rail.

This repository carries the two halves of that job:

| | what it is | who it is for |
| --- | --- | --- |
| [`lab/`](lab/) | `dsh-ui-lab` — a headless Chromium harness that loads a DSH surface at real device proportions and writes screenshots, measurements and matrix reports, with a pixel baseline gate | anyone checking whether a UI change moved something on a phone |
| [`packs/`](packs/) | opt-in adaptation packs loaded by `dsh-mobile`'s `extensions/` directory — 44px touch floor, rail clearance, floating-widget clearance | people whose plugin panels need to survive a phone today |

They are independent. The lab needs no pack; the packs need no lab. Together they close the loop: the lab says *where* it breaks, a pack fixes it, the lab's baseline says whether the fix held.

```
   dsh-ui-lab  ──measure──▶  DSH surface (390×844, coarse pointer)
                                   ▲
        packs  ──install────▶  $DSH_HOME/mobile-access/extensions/<id>/
```

---

## Quick start — the lab

Node 20+, and a Chromium for Playwright.

```bash
cd lab
npm ci
npx playwright install chromium

npx . list                                        # 10 proportions: 6 phones, 4 tablets
npx . shot --surface mobile --profile phone-390x844 --open Todos
npx . matrix --surface mobile --class phone --open Todos --out .shots/phone
```

`--surface mobile` drives the real `dsh-mobile` gateway (it pairs a device once and caches the state). `--surface web` opens stock DSH web at the same proportions. `--open <label>` clicks a sidebar entry first, so the capture is the panel — not the welcome screen.

No DSH at all? The repository ships a fixture:

```bash
cd lab
npx . shot --surface web --profile phone-390x844 \
  --url "file://$PWD/test/fixtures/desktop.html" --out .shots/fixture
```

A baseline turns a matrix into a gate:

```bash
npx . matrix --surface mobile --class phone --open Todos --out .shots/run --save-baseline ../.baselines/todos
npx . matrix --surface mobile --class phone --open Todos --out .shots/run --baseline ../.baselines/todos
# exit 1 when the pixels moved past the tolerance
```

Full documentation: [`lab/README.md`](lab/README.md) (English) · [`lab/README.zh.md`](lab/README.zh.md) (中文).

---

## Quick start — the packs

The packs are loaded by **dsh-mobile**, through its public `extensions/` interface — no fork, no patching of installed files, and removing a directory unloads it within two seconds.

```bash
node tools/check-pack.mjs                 # validate every pack (static + fixture + opt-out)
node tools/install.mjs pack core-panel --dry-run
node tools/install.mjs pack core-panel --yes
```

| pack | scope | what it does |
| --- | --- | --- |
| `core-panel` | plugin panels | 44px floor inside a panel, width reclaim, clearance for the 56px collapsed rail |
| `shell-chrome` | the shell itself | the panel's "back to conversation" bar and the open drawer's rows |
| `floaters` | floating widgets | measures bottom-right fixed widgets and reserves their height, so the last rows are no longer hidden underneath |

Every pack is `experimental` and off by default; `catalog.json` records what has actually been verified and what has not. Each one honours an opt-out (`<html data-dsh-adapt="off">`) and is checked against a fixture before it ships.

Full documentation: [`docs/收录标准.md`](docs/收录标准.md) · [`packs/core-panel/README.md`](packs/core-panel/README.md) · [`docs/phone-instance.md`](docs/phone-instance.md).

---

## What has actually been verified

Honesty matters more than a green checkmark here, so the state is written down in [`catalog.json`](catalog.json)'s `evidence` field per pack:

| | state |
| --- | --- |
| Fixture measurement (390×844 coarse pointer, 1280×800 fine pointer zero-regression, opt-out) | ✅ every pack, `tools/check-pack.mjs` |
| Real runtime on **dsh-mobile 0.7.1** | ✅ every pack, measured on a real gateway |
| Real runtime on **0.8.0** | ❌ only a text-level check of `lib/mobile-layout.js` |
| A physical device | ❌ never — no thumb, no soft keyboard, no real `env(safe-area-inset-*)` |

The packs therefore stay `experimental` (not `candidate`). If you run them on a device, the baseline gate in the lab is the fastest way to tell whether a difference came from the pack.

The packs also depend on class names that `dsh-mobile` does not promise to keep (`.dshm-shell`, `.dshm-main`, `.dshm-drawer`, …). `catalog.json` lists the markers each pack touches precisely so a break is traceable, but a `dsh-mobile` redesign can still break them. This is an unofficial compatibility layer, not something the dsh-mobile maintainers ship or support.

## Contributing

Adding a proportion is one entry in [`lab/profiles/profiles.json`](lab/profiles/profiles.json). Adding a pack starts from [`packs/_template/`](packs/_template/) and has to pass `node tools/check-pack.mjs packs/<id>`; the rules are in [`docs/收录标准.md`](docs/收录标准.md). See [`lab/CONTRIBUTING.md`](lab/CONTRIBUTING.md) for the test and commit conventions.

## License

MIT — see [LICENSE](LICENSE).
