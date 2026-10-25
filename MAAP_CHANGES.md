# Changes from upstream

Upstream: https://github.com/JPL-WVCC/CrIS_VIIRS_collocation-master, branch `master` at commit 77f84fe4e79a7308e3995dc5a69d0e53d60898ac (2026-04-08), the MEaSUREs WVCC CrIS/VIIRS collocation library (Wang et al. 2016 method; code by Qing Yue and Lei Pan, JPL, with documentation by pymonger). Upstream has no tags, releases or LICENSE file; all credit stays with the original authors.

This copy starts from upstream's file tree at 77f84fe without its git history. Upstream has no `.github/` folder, so nothing was left out. It is unchanged except for the items below. The production runner, `matchup/parallel_run_matchup.py` in https://github.com/JPL-WVCC/matchup_pge (commit 77def16), is used unchanged.

## 2026-10-25: `np.int` replaced by `int`

`geo_QY.py` computed the search window size with `.astype(np.int)` in `find_match_index_QY` (the function the runner uses) and in the older `find_match_index`. numpy removed the `np.int` alias in 1.24, so the matching stopped with an `AttributeError` on any current numpy, including the numpy 1.24.4 that `pip install -r requirements.txt` picks on Python 3.8. Both now use the builtin `int`, which is what `np.int` always meant. Results do not change.

## 2026-10-25: per-FOV match indexes kept as object arrays

`match_cris_viirs_QY` passed the lists of per-FOV index arrays to `np.array()`. The arrays differ in length (260 to 1,450 VIIRS pixels per FOV in the J1 test granule), so numpy 1.23 and older made an object array with a `VisibleDeprecationWarning`, and numpy 1.24 and newer stop with `ValueError: ... inhomogeneous shape`. A small helper, `list_to_object_array`, now builds the same object array explicitly, and the "no co-location" test counts the matched pixels directly (it returns early in the same cases as before). The arrays and their order are the same, so results do not change.

## 2026-10-25: numpy 2 allowed

`requirements.txt` dropped the `<2.0` cap on numpy, and the README says that numpy 1.24 and newer, including numpy 2, work.

## Known issues, not changed

- **Variable names are swapped in the product.** `code_test_QY.py:186-191` writes the VIIRS along-track line (in the stacked granules) into `number_of_pixels`, whose long name says "imager cross-track index", and the cross-track pixel into `number_of_lines`. The official GES DISC files have the same layout, so this is kept to stay identical to them. Readers should take `number_of_pixels` as the line and `number_of_lines` as the pixel. In the J1 test granule `number_of_pixels` runs from 3176 to 6519 (of 9,664 stacked lines) and `number_of_lines` from 319 to 2881 (of 3,200 pixels).
- **A failed granule still exits 0.** matchup_pge's `parallel_run_matchup.py` runs each CrIS granule in a `multiprocessing.Process` and does not check its exit code, so a crash in this library leaves an empty product folder and the runner still exits 0. `code_test_QY.py:159` also calls `sys.exit(0)` when no co-location is found. Check that the product `.nc` file exists rather than relying on the exit status.

## Check

On 2026-10-25, from a fresh `git clone` of this repository, in a new conda env with Python 3.12.15 and `pip install -r requirements.txt` (numpy 2.5.4, netCDF4 1.7.5.1.post1, pykdtree 1.4.3, cftime 1.6.6.1), with matchup_pge at 77def16 unchanged and this repository on `PYTHONPATH`, from `matchup_pge/matchup/` (the input and output folders are relative to it and must end in `/`):

```sh
PYTHONPATH=/path/to/CrIS_VIIRS_collocation-master \
python parallel_run_matchup.py --y 2025 --m 12 --d1 31 --d2 31 --sp J1 \
  --pr out/ --cr cris/ --vr viirs/ --c 2
```

The inputs were the NOAA-20 CrIS L1B granule `SNDR.J1.CRIS.20251231T1936.m06.g197.L1B.std.v03_08.G.260101021810.nc` (GES DISC, `SNDRJ1CrISL1B` v3) and the five `VJ103MOD` granules from 19:24 to 19:48 UTC on day 365 (LAADS, collection 5201), laid out as `cris/2025/12/31/crisl1b/` and `viirs/2025/365/`. The product `IND_CrIS_VIIRSMOD_SNDR.J1.20251231T1936.g197.nc` is identical to the official GES DISC `J1_CrIS_VIIRS750m_IND` V1 granule: all 12,150 `FOVCount_ImagerPixel` values, all 5,605,455 values of `number_of_lines` and `number_of_pixels`, and every global attribute except `PRODUCTIONDATE`. It took 17 s with no network and peaked at 3.9 GB.
