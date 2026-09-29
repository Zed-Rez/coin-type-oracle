# coin-type-oracle

Look up an ancient coin type by its catalogue number and get back everything the corpus knows about it:
the type record, the concordance to other catalogues, equivalent types, hoards and single finds, metal assays,
and every known specimen (exemplum) with a link to the holding museum or sale.

```
$ oracle ric 3.232
oracle: 5 type(s) for ric 3.232
  #  type                         authority        mint  denomination  years     coins
  1  RIC III Antoninus Pius 232   Antoninus Pius   Rome  Denarius      153..154  16
  2  RIC III Commodus 232         Commodus         Rome  Denarius      192..192  0
  3  RIC III Marcus Aurelius 232  Marcus Aurelius  Rome  Denarius      170..171  0
  4  RIC III Antoninus Pius 232A  Antoninus Pius   Rome  Denarius      153..154  0
  5  RIC III Antoninus Pius 232a  Antoninus Pius   Rome  Denarius      153..154  0
  ...
```

It is the type oracle from the Roman coin ViT project packaged as a single [DuckDB](https://duckdb.org) file plus one
Python script.

## Download

The database is too large for a git commit, so it ships as a release asset:

**[`coin-type-oracle.zip`](../../releases/latest)** (Releases → latest)

| file | what it is |
|---|---|
| `oracle.duckdb` | the database, 2.0 GB unzipped (the zip is 576 MB), DuckDB storage format of duckdb 1.5.x |
| `oracle_db.py` | lookup CLI (and the builder used on the source machine) |
| `oracle` | shell launcher |
| `requirements.txt` | `duckdb==1.5.6` |

## Setup

Needs Python 3.9 or later and one package.

```bash
unzip coin-type-oracle.zip -d coin-type-oracle
cd coin-type-oracle
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
.venv/bin/python oracle_db.py ric 3.232
```

To call it as `oracle` from anywhere, put the launcher on your `PATH`; it runs the `.venv` next to it:

```bash
chmod +x oracle
ln -s "$PWD/oracle" ~/.local/bin/oracle
```

`oracle_db.py` finds the database in this order: `$ORACLE_DB`, then `oracle.duckdb` next to the script, then
`~/Desktop/coins/cleaned/oracle.duckdb`. Keep the DuckDB version at 1.5.x: other major versions may not open
the file.

## Usage

```bash
oracle ric 3.232               # every type RIC III 232 can mean, letter variants (232A, 232a) included
oracle ric 3.232 commodus      # narrow by ruler or mint (matched against authority, mint, OCRE id, label)
oracle ric 2.1 232             # volume and number as separate tokens (RIC II.1, second edition)
oracle ric 3.232 --exact       # 232 only, no letter variants
oracle rpc 2.2311              # Roman Provincial Coinage volume.number
oracle rpc id 7958             # RPC Online's own type id
oracle rrc 238/1               # Crawford, Roman Republican Coinage
oracle sc 1.232                # Seleucid Coins; also price, rsc, bmc, sydenham, sng, cohen ...
oracle --uri http://numismatics.org/ocre/id/ric.3.ant.232
oracle --coin ans/1911.23.225  # one specimen: every field, images, references, verbatim source properties
```

| flag | effect |
|---|---|
| `--all` | list every exemplum (default cap 25 per type) |
| `--full` | full record and all verbatim source fields for each exemplum, raw CHRE hoard rows |
| `--exact` | no letter variants |
| `--json` | machine-readable output |

Volumes can be given in Roman or Arabic numerals (`ric III.232` = `ric 3.232`). Any system in the concordance
table works, keyed by its lowercase name without spaces.

## What each result contains

For every matching type:

- **Type record**: authority, mint, denomination, material, region, issuer, portrait, dates (BC negative),
  obverse and reverse legends and designs, corpus counts.
- **RPC type record and specimens** for provincial types: province, city, reign, inscriptions with edition and
  translation, average weight, diameter and axis, cited literature.
- **Concordance**: the same type in other catalogues (RSC, Sydenham, SNG, BMC, Meshorer ...), with how many
  coins support each equation.
- **Equivalence class**: letter variants, subtypes, denomination variants and superseded editions of the
  same type.
- **Finds**: hoards containing the type (CHRE, grouped by hoard, with coordinates, quantity, closing date and
  hoard size), Republican hoards (CHRR), single finds (AFE).
- **Metal assays** joined to the type: ADSL denarii and provincial silver, GFZ lead isotopes, RRC fineness.
- **Exempla**: every coin in the corpus linked to the type, with museum or sale URL, weight, diameter, die
  axis, findspot, licence, image size, and how it was linked (catalogue link or parsed reference).
- **Unlinked citations**: coins that cite the number but never resolved to a type (ambiguous ruler, missing
  volume, RPC type not crawled). Check these by hand.

## What is in the database

| table | rows | contents |
|---|---|---|
| `types` | 167,977 | one row per type across OCRE/RIC, RPC, CRRO/RRC, SCO, PELLA, IRIS, Corpus Nummorum and others |
| `type_references` | 179,045 | concordance of every type to other catalogue numbers |
| `type_keys` | 248,641 | normalised lookup keys (system, volume, number) |
| `type_equiv` | 167,977 | equivalence classes |
| `coins` | 843,566 | deduplicated specimens from about 50 collections, dealers and datasets |
| `properties` | 40.9 M | every source field for every coin, verbatim, long format |
| `images` | 1.2 M | image files per coin (paths only, no pixels) |
| `coin_references`, `coin_keys` | 183,526 | catalogue citations parsed from coin records |
| `chre_coins`, `chre_hoards`, `type_findspots` | 300k / 18.7k / 140k | Coin Hoards of the Roman Empire |
| `chrr_hoards`, `chrr_contents` | 694 / 32k | Coin Hoards of the Roman Republic |
| `afe_coins`, `afe_places` | 19k / 949 | Antike Fundmünzen in Europa single finds |
| `rpc_types`, `rpc_specimens` | 13.4k / 46.6k | Roman Provincial Coinage Online |
| `flame_*` | | FLAME late-Roman coin finds and print runs |
| `assay_adsl`, `assay_gfz`, `assay_rrc` | 381 / 188 / 480 | metal analyses joined to types |

The view `coin_full` joins each coin to its type. You can query everything directly:

```bash
.venv/bin/python -c "import duckdb; c = duckdb.connect('oracle.duckdb', read_only=True); \
print(c.sql(\"select type_number, n_coins from types where corpus = 'ocre' order by n_coins desc limit 10\"))"
```

## Limits

- **RPC coverage** is what the crawl reached: 13,400 types, mostly volumes II, III and IX. Most RPC I types
  are absent, so `oracle rpc 1.x` usually returns only the coins that cite the number.
- **Linking is partial**: 359,289 of 843,566 coins (43%) are linked to a type, and 27,923 of 56,143 RIC types
  have at least one linked coin.
- **Image paths** point into the source machine's `cleaned/` folder. The images are not in the zip. Use the
  record URLs to see them.
- **Snapshot**: built 2026-09-29 from the corpus database of 2026-09-21.
- **The `type_finds` summary** is older than the hoard tables and can undercount hoards and coordinates. Trust
  the hoard table.

## Rebuilding

`oracle_db.py build` regenerates `oracle.duckdb` from `coins.sqlite` and the side files under
`~/Desktop/coins/` (`COINS_HOME` overrides this). It only runs on the machine that holds the corpus. The build
takes about a minute.

## Data and licences

The records come from museum catalogues, Nomisma.org, OCRE, CRRO, RPC Online, CHRE, CHRR, AFE, FLAME, dealer
archives and published datasets, each under its own terms (several are CC BY-NC-SA). This repository is
private for that reason. Check the licence column of each record before any reuse.
