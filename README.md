# Amazon geoglyphs: can satellite data tell known sites from random places?

A learning project in Google Earth Engine and Python. I wanted to know if free satellite data at 10 to 30 m resolution can tell where a known Amazon geoglyph is from similar random cleared land, when the model is tested on areas it hasn't seen.

**It is not a site detector and it doesn't find new sites.**

**Short version: it works a little, not well.**
- Optical features alone give an AUC of about 0.62 (0.50 is chance). Two separate draws of random places give similar numbers, and shuffled-label controls sit at 0.50.
- A small CNN does no better.
- Elevation on its own is as good as all the optical features together (0.615 against 0.619). Both together give about 0.72. I read that as where sites sit in the landscape, not the satellite seeing the earthworks. I can't rule out survey bias (see Limitations).

**What was planned and what wasn't.** Section 1 of the results was planned. The CNN and everything after it I added after seeing the first results, so it's exploratory. I report every variant I ran, including the ones that didn't help.

**Checking the numbers.** Every number here was copied from a saved log, not from memory: `run_log_draw2.txt` (the notebook as I ran it) and `run_log_draw1_excerpt.txt` (the first run).

## Data

**Catalogue.** The data is James Q. Jacobs' Amazon geoglyph database ([Hundreds of Geoglyphs Discovered in the Amazon](https://www.jqjacobs.net/archaeology/geoglyph.html)). He compiles it from Google Earth and other satellite imagery, aerial photos, published databases and papers, and keeps updating it. I downloaded it on 5th august 2026 and saved it as `amazon_geoglyphs.xlsx`: 1,490 rows of geoglyphs and similar earthworks, with coordinates, elevation, shape (`form`), one or two widths and a `source` field. The live file changes, so a new download will have more rows than mine. The positions come from others and I didn't check how accurate they are.

**The catalogue is not in this repo.** The page says "All Rights Reserved" and gives no licence for the data, so I don't redistribute the spreadsheet or any file with its coordinates. Download the original from the page above. My cleaned and selected copy (`geoglyphs_selected_v2026-08-18.csv`) is not included either; the steps that made it are in the Cleaning and Picking sections below, and `01_clean_and_select.ipynb` repeats them on a fresh download.

**Cleaning.** Done in `01_clean_and_select.ipynb`. Counts are from my snapshot.

| Step | Rows left |
|---|---|
| Read the spreadsheet | 1,490 |
| Removed 8 blank rows (no code, so no coordinates) | 1,482 |
| Removed 2 features with no width (saved in `dropped_no_size.csv`) | 1,480 |
| One duplicated code, two image dates: kept the newest | 1,479 |
| Five pairs with identical coordinates: kept the newest image | 1,474 |

Where the second width (`b width`) was missing I set it equal to the first and flagged it `b_filled`. That's 707 of the 1,474. The cleaned table is `geoglyphs_clean_v2026-08-18.csv`.

**Picking the 1,003 positives.** From the 1,474 I dropped the 171 `zanja` features (very large ditched enclosures, a different class) and anything with a shorter side under 90 m. For rows with a filled second width, the 90 m rule only checks the first width. That leaves 1,003, saved as `geoglyphs_selected_v2026-08-18.csv`. **So everything here is about enclosures 90 m or bigger (widths 90 to 604 m), not small geoglyphs.**

## How the test works

**Imagery.** Sentinel-2 surface reflectance, June to September 2025, cloud-masked with Cloud Score+ (clear-sky probability of at least 0.60), median composite. I picked 2025 because it had the most clear observations: a median of 26 per site (2023: 19, 2024: 19, 2026: 17), and every site had at least 5.

**Windows and features.** For each site I take an inside disc (radius half the first width) and a ring (radius 0.75 to 1.5 times the first width). Features are the mean of NDVI, NDMI, BSI and SWIR1 inside, the same in the ring, the difference between them, and the NDVI standard deviation inside. Nine features in total.

**Negatives.** 6,000 random points on cleared land: NDVI between 0.356 and 0.776 (the 5th and 95th percentiles of NDVI at the positives), within 20 km of the positives, and at least 1 km from known features.
- Draw 1: "known features" meant all 1,474 in the cleaned catalogue.
- Draw 2: the cleaned table wasn't loaded (the notebook printed a warning), so I only kept negatives 1 km from the 1,003 selected positives. Some draw-2 negatives may be within 1 km of one of the other 471 known features.

Each negative borrows a first width at random from the positives, so both groups use the same window sizes (mean about 178 m).

**Matching.** Each positive gets one negative, chosen by nearest neighbour on ring NDVI and ring SWIR1 (standardised, without replacement, positives in random order, seed 42). That gives 2,006 rows. Matching on the surroundings removes the easy "different kind of land" signal, so the test is hard on purpose. Draw 2, ring means (negatives / positives):

| Ring mean | Before matching | After matching |
|---|---|---|
| NDVI | 0.598 / 0.567 | 0.567 / 0.567 |
| SWIR1 | 0.261 / 0.295 | 0.295 / 0.295 |
| BSI | 0.008 / 0.043 | 0.044 / 0.043 |
| NDMI | 0.027 / -0.008 | -0.010 / -0.008 |

I didn't match on BSI or NDMI, but they came into balance too.

**Validation.** Spatial blocks of 0.25 degrees (about 28 km, 152 blocks in draw 2). 20 random splits that hold out whole blocks (30% of blocks each, seed 42). The metric is ROC AUC: the mean, with the lowest and highest of the 20 splits in brackets. Those ranges aren't confidence intervals, because the splits overlap. Random forest: 300 trees. Logistic regression: standardised inputs. Controls: shuffled labels, and an NDVI-only baseline.

## Results

### 1. Planned analysis, two draws of negatives

The negatives from the first draw weren't saved, and I couldn't reproduce them when I tried to extract again (the positives matched, the negatives didn't). So the draw-1 numbers are as I got them and can't be re-checked. The notebook here redraws the negatives, saves them before extraction, and gives the draw-2 column.

| Model (spatial-block AUC) | Draw 1 | Draw 2 |
|---|---|---|
| NDVI inside only (logistic) | 0.527 | 0.514 |
| Logistic regression, 9 features | 0.615 | 0.613 |
| Random forest, 9 features | 0.636 (0.575 to 0.681) | 0.619 (0.583 to 0.648) |
| Random forest, rows with a filled second width removed | 0.648 | 0.637 |
| Random forest, shuffled labels | 0.505 | 0.502 |

Everything moved by about 0.01 to 0.02 between draws, which is inside the spread between splits.

The draws differ in more than the seed: draw 1 kept negatives away from all 1,474 known features, draw 2 only from the 1,003 selected ones. Unselected known sites sitting in the negatives would, if anything, lower the AUC. Draw 2 is a bit lower, but the gap is inside the spread between splits, so I don't read anything into it.

A single random 70/30 split that ignores spatial blocks gives slightly higher AUCs (draw 2: random forest 0.636, logistic 0.615, NDVI-only 0.545. Draw 1: random forest 0.649, logistic 0.632, NDVI-only 0.554). I only use the spatial-block numbers. For scale: on that draw-2 split, at a 0.5 cutoff, the random forest gets 59% of the test rows right (precision 0.58, recall 0.62).

### 2. Small CNN (exploratory)

I fixed the settings before running, but added it after seeing the random forest results. 33 x 33 pixel patches at 20 m (660 m across; bands B4, B8, B11), three convolution blocks, 20 epochs, no tuning, same rows, blocks and seeds, all 2,006 rows.

| Model | Spatial-block AUC |
|---|---|
| CNN | 0.644 (0.602 to 0.680) |
| CNN, shuffled labels (5 splits) | 0.501 (0.489 to 0.519) |

About the same as the random forest on this draw (0.619). Not clearly better, because the ranges overlap almost completely.

### 3. Other checks (exploratory, all added after sections 1 and 2)

I treat differences under about 0.03 AUC as noise.

**Enclosure size.** I split the positives into thirds by width and scored each third against the negatives in the same width range.

| Width | Positives / negatives | AUC |
|---|---|---|
| 90 to 137 m | 335 / 361 | 0.574 (0.535 to 0.630) |
| 137 to 192 m | 337 / 317 | 0.586 (0.506 to 0.661) |
| 192 to 604 m | 331 / 325 | 0.642 (0.553 to 0.698) |

AUC goes up with size, which fits a resolution limit. But the ranges overlap a lot, and size is tangled up with window size and location, so it only hints.

**Feature sets**, all on the same 2,006 rows and splits.
- Wet season: December 2024 to March 2025 (no rows lost).
- Radar: Sentinel-1 dry-season median of VV, VH and VV minus VH (415 scenes, ascending and descending mixed).
- Terrain: SRTM elevation and slope, each as inside, ring and difference.

| Features | AUC |
|---|---|
| Optical, dry season (the reference) | 0.619 (0.583 to 0.648) |
| Wet season only | 0.580 (0.535 to 0.621) |
| Optical dry + wet | 0.653 (0.617 to 0.692) |
| Radar only | 0.548 (0.495 to 0.587) |
| Optical + radar | 0.641 (0.606 to 0.677) |
| Terrain only | 0.662 (0.609 to 0.700) |
| Optical + terrain | 0.716 (0.683 to 0.751) |
| Optical + radar + terrain | 0.724 (0.695 to 0.768) |

Controls for terrain: shuffled labels with optical + terrain gave 0.507 (0.486 to 0.524, 5 splits). Elevation alone gave 0.615 (0.581 to 0.646). Slope alone gave 0.555 (0.514 to 0.591). The dry + wet run, the wet-only run and the size bins have no shuffled control of their own.

**What it adds up to.**
- Optical features give a weak but real, stable signal.
- A wet-season composite adds about 0.03, which is borderline. Radar adds nothing clear.
- Elevation alone matches all the optical features, and they aren't redundant (0.615 and 0.619 alone, 0.716 together). Terrain is the one big gain: about +0.10 over optical, with ranges that don't overlap.

## Limitations

- **Not detection.** AUC here is how well known sites separate from matched random cleared land. It says nothing about finding unknown sites. I had no independent validation set, like an unbiased survey or LiDAR.
- **Size and class.** Only enclosures at least 90 m on the shorter side, and not `zanja`. The results don't cover smaller geoglyphs.
- **Survey bias.** The positives are catalogued sites, many found on Google Earth, which leans toward cleared, surveyed, easy-to-reach land. The terrain gain could partly be where people looked, not where geoglyphs were built. I can't separate the two. SRTM is also a 30 m surface model that includes vegetation height.
- **Negatives aren't verified empty.** Random cleared points could contain undiscovered geoglyphs. In draw 1 they're at least 1 km from all 1,474 known features. In draw 2 only from the 1,003 selected positives, so some may be near a known but unselected feature.
- **Position and date.** Coordinates come from a compiled, regularly updated database (Jacobs) and I didn't assess their accuracy. The composite is 2025, but features were found on older imagery (`img_date`), so land cover may have changed since.
- **Resolution is a guess.** Where dimensions are reported, the ditch is about one Sentinel-2 pixel wide (de Souza et al. 2018 give ditches 1 to 3 m deep in the Upper Tapajos Basin, and 8 to 10 m wide at the one site where width is given; I didn't find figures for the Acre sample). My inside window averages the whole enclosure, which may water the ditch down. The size result is only indirect support. I haven't shown resolution is the cause.
- **Many comparisons.** Section 3 is about a dozen variants run after seeing results. A +0.03 in one of them can be chance. Only the terrain gain is big compared with the spread between splits.
- **Splits overlap.** The 20 splits aren't independent, so the ranges are descriptive only.
- **One year, one region.** Composite is June to September 2025. Earth Engine data gets reprocessed, so a re-run later may not give exactly the same numbers.
- **Small CNN, no tuning.** It says nothing about deep learning in general.
- **Not one clean run.** I built and ran the notebook in pieces over several sessions. The notebook saves partial CSVs as it goes, so you can resume. I haven't re-run it start to finish in one go.
- **Bugs I hit and fixed** (listed so nothing is hidden):
  - A global variable `cols` got overwritten by a loop and broke one extraction.
  - I dropped an early terrain feature (1 km local relief) because it made Earth Engine far too slow, before seeing any result. The terrain cell in `run_log_draw2.txt` still has the old relief code below the fixed lines. It did nothing, because the extraction had already finished. I left that cell as I ran it.
  - The CNN cell prints a hard-coded comparison number (0.636, which is draw 1's value, not draw 2's 0.619). I left that cell as I ran it.

## Running it

1. Download the catalogue from the page above and save it as `amazon_geoglyphs.xlsx`.
2. Run `01_clean_and_select.ipynb`. It writes `geoglyphs_clean_v2026-08-18.csv` and `geoglyphs_selected_v2026-08-18.csv`.
3. Open `geoglyph_analysis.ipynb`. It's written for Colab and needs `earthengine-api`, `geemap`, `scikit-learn`, `pandas`, `numpy` and `tensorflow`.
4. Put your own Google Cloud project ID (registered for Earth Engine) in the first code cell. Don't commit it.
5. Upload both CSVs from step 2. The clean one is optional: it keeps negatives away from all 1,474 known features, and without it the notebook falls back to the 1,003 selected ones, like draw 2.
6. Run top to bottom. Extraction is slow (tens of minutes per block) and saves partial CSVs so you can resume.

## Data, ethics and AI

The positives are the locations in Jacobs' catalogue, which compiles public sources and imagery. Nothing from the catalogue is redistributed here. The negatives are random points. There are no coordinates of previously undocumented sites in this repo, and I don't use the model's outputs to locate any.

I used AI (Claude) to help write parts of the notebook and this README while I was learning Earth Engine and spatial validation. I ran everything and I stand behind the content.

## Citing and licence
Code: MIT. The catalogue data belongs to Jacobs and isn’t covered by this licence.

Data: Jacobs, J. Q. (2010 onward). *Hundreds of Geoglyphs Discovered in the Amazon* [web page and database]. https://www.jqjacobs.net/archaeology/geoglyph.html. Accessed 5th August 2026.

Also: de Souza, J. G. et al. (2018). Pre-Columbian earth-builders settled along the entire southern rim of the Amazon. *Nature Communications* 9, 1125.

