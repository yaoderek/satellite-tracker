# satsim/hifi: freshest elements, uncertainty estimates and 72 h ephemerides

Built 2026-10-17 at about 19:50 UTC. Ephemeris start t0 = **2026-10-17T19:53:00Z**, span 72 h.
Observer: Bellevue, WA, 47.6101 N, -122.2015 E, 50 m (WGS84).
Everything here was propagated with **SGP4** (the `sgp4` 2.27 library, frame conversions with skyfield 1.55 and DE421).
"High fidelity" here means the freshest public elements, precise frame handling, and an honest error estimate.
It does **not** mean precision orbit determination. SGP4 mean elements set the accuracy floor.

## 1. Sources and selection (`summary.*`: `source`, `source_kind`, `epoch_utc`, `age_days`, `n_candidates`)
| kind | source | used for |
|---|---|---|
| SUP | CelesTrak **SupGP** (operator or laser-ranging-derived OMM: cpf, gps, glonass, intelsat, ses, iridium, eumetsat, starlink, oneweb, telesat) | 25 objects (23 GLONASS from GLONASS-RE, 2 from CPF laser ranging incl. Etalon 2) |
| GP | CelesTrak **GP** in OMM/JSON (groups active, military, gps-ops, glo-ops, galileo, beidou, geo, analyst, last-30-days) | 624 objects |
| MC | Mike McCants amateur TLEs `classfd.tle` and `inttles.tle` (mmccants.org, fetched today) | 420 objects |
| SS | SeeSat-L archive Sep–Oct 2026 (satobs.org), all posted TLEs parsed | 0 objects |

* All 649 public objects were present in the bulk GP group downloads, so I didn't run 649 separate `CATNR=` queries. The bulk groups serve the same GP records and avoid CelesTrak's rate limit.
* For each object, every candidate element set was scored by its *predicted along-track 1σ now* (the model in §2). The lowest score was kept, which in practice is "freshest, with SupGP preferred over GP preferred over amateur."
* **SeeSat-L:** I found 10 element sets in 167 Sep/Oct posts. They were all for analyst or unknown IDs (99999, 99151, 70195 "USSF-259") that don't match a catalogued object in this set. None were newer than McCants for a set member, so none were used. They were not matched to unpublished Starshield or temp IDs on purpose.
* The McCants files downloaded today are byte-identical to `../classfd.tle` and `../inttles.tle`. The last McCants update was 2026-10-15 (classfd) and 2026-10-17 (inttles).
* `elements_raw` holds the exact OMM JSON or TLE that was used. `globe3d.html` uses TLEs exported from those same records.
* 39 McCants sets have an epoch slightly **in the future** (`future_epoch`). This is a normal amateur practice. Age is treated as |age|.

## 2. Uncertainty model (`sig_{along,cross,radial}_{now,72h}_km` and per-sample `sig_*` in the ephemerides)
For each component c: σc(t) = sqrt(s0c² + (ratec·|age|)² + (qc·age²)²), with age in days since epoch. The per-object coefficients are stored in the `s0_*`, `rate_*` and `q_along` columns.
These values are meant as a 1σ-ish *typical* error. They are not a covariance, because no public covariance exists for GP or TLE data.

**Literature basis:**
* Hoots & Roehrich, *Spacetrack Report #3* (1980). Vallado et al., "Revisiting Spacetrack Report #3", AIAA 2006-6753.
* Kelso, "Validation of SGP4 and IS-GPS-200D against GPS precision ephemerides", AAS 07-127 (2007). GP errors at epoch are about 1 km to a few km and grow by about 1 km/day to several km/day.
* Flohrer, Krag & Klinkrad, "Assessment and categorization of TLE orbit errors for the US SSN catalogue", AMOS 2008. Error is dominated by along-track. Radial and cross-track are about 0.1–1 km in LEO. Errors are larger for GEO and HEO.
* Vallado & Crawford, "SGP4 orbit determination", AIAA 2008-6770.
* Errors grow fastest for low-perigee, high-drag objects, especially near solar maximum, and for any object that maneuvers.

**Empirical calibration (`calib_summary.csv`, `calib_gp_vs_supgp.csv`):** 13,331 objects have both GP and SupGP sets. I propagated each GP set to its SupGP epoch and differenced the two in radial/in-track/cross-track (RIC) coordinates. Median |error| as a function of GP age:

| regime | age 0–0.5 d | 0.5–1 d | 1–2 d | 2–4 d | 4–8 d |
|---|---|---|---|---|---|
| LEO Iridium (780 km) along | 0.30 | 1.0 | – | – | – |
| LEO OneWeb (1200 km) along | 0.86 | 0.88 | 2.2 | – | – |
| LEO Starlink (~350–550 km, maneuvering) along | 2.2 | 7.3 | 21.6 | ~1000+ (n=6) | – |
| LEO, all, cross / radial | 0.11 / 0.15 | 0.11 / 0.19 | 0.15 / 0.27 | | |
| MEO (GPS/GLONASS) along | 0.35 | 0.22 | 0.70 | 0.78 | 1.4 |
| GEO (Intelsat/SES) along | 0.82 | 1.13 | 1.18 | 1.6 | 9.8 (p90 30) |

The model coefficients were set from this table. GP LEO coefficients by perigee band (s0 along / rate along):

| perigee | s0 along | rate along | quadratic term |
|---|---|---|---|
| >700 km | 0.8 km | 1 km/d | none |
| 500–700 km | 1.5 km | 3 km/d | none |
| 400–500 km | 2 km | 8 km/d | q = 1 km/d² |
| <400 km | 3 km | 15 km/d | q = 3 km/d² |

GP for other regimes: MEO 0.5 km + 0.4 km/d, GEO 2 km + 1.6 km/d, HEO 5 km + 4 km/d. HEO is not calibrated because no reference data was available.
For SupGP, s0 comes from its fit RMS and the growth rate is halved.

Other empirical files:
* `empirical_checks.csv`: GP vs SupGP for the set members themselves (25 MEO: median along 0.55 km at about 1 d), plus older-set vs newer-set checks.
* There were few old-vs-new pairs, because the base CSV was fetched earlier today. CelesTrak serves no element history without Space-Track, which I did not use.

**Amateur (McCants) elements have no public truth to test against.** Their model is a judgment call based on the GP model:
* LEO: s0 is the larger of 3×GP or 5/2/1 km (along/cross/radial). Growth is the larger of 2×GP or 8 km/d.
* GEO: 20 km + 5 km/d. HEO: 20 km + 20 km/d. MEO: 15 km + 5 km/d.
* Classified objects **maneuver without notice**, and no model captures that.

## 3. Quality tiers (`quality_tier`, by along-track 1σ now)
| tier | σ along now | count | GP | SupGP | McCants |
|---|---|---|---|---|---|
| A | <1 km | 169 | 144 | 25 | 0 |
| B | 1–5 km | 398 | 398 | 0 | 0 |
| C | 5–25 km | 143 | 80 | 0 | 63 |
| D | 25–200 km | 123 | 2 | 0 | 121 |
| E | >200 km or stale | 236 | 0 | 0 | 236 |

Flags:
* `stale_flag` (age >30 d): 218 objects.
* `position_unreliable`: 63 objects. This covers age >180 d, σ more than ¼ of an orbit, or SGP4 failing. For these, the computed position is effectively unknown, and their passes are left out of the globe list.
* SGP4 returned errors over the whole window (decayed per SGP4, `sgp4_err`≠0) for 6 objects: USA 224, USA 312, USA 313, USA 335, Limasat and Ofeq 19 r. Their elements are 1–3.4 years old.

Typical medians (σ along / cross / radial now, km; and timing σ now):
* GP LEO: 1.8 / 0.15 / 0.15, about 0.24 s. Grows to about 10 km along after 72 h.
* GP GEO: 2.1 / 1.0 / 0.5. GP MEO: 0.6. GP HEO: 5.9.
* SupGP GLONASS: 0.4 km.
* McCants GEO: about 20 km. McCants HEO: about 21 km.
* McCants LEO: about 300 km. The median age is 36 d, and many sets are months to years old.

## 4. Files
* `summary.parquet` / `summary.csv`: one row per object (1,069 rows). Contains the elements (inc, ecc, RAAN, argp, M, mean motion), period, semi-major axis, apogee/perigee, BSTAR and ndot. `decay_km_day` is da/dt from ndot, i.e. -2/3·a·ṅ/n. Also has `supgp_rms_km`, the σ columns and coefficients, `timing_sigma_*_s` (σ along / speed), tier, flags and the raw elements.
* `ephem_60s_72h.parquet` (zstd, about 750 MB): 4,321 epochs × 1,069 objects = 4,619,149 rows. Columns:
  * `norad`, `time_utc`
  * `teme_*`: SGP4 native output
  * `gcrs_*`: TEME→GCRS via skyfield, using the IAU 2000A/2006 path
  * `ecef_*`: ITRS, including polar motion, with velocity including ω×r
  * Position in km, velocity in km/s
  * `sunlit`: whether the line from satellite to Sun clears the Earth sphere (R = 6378.137 km). This is a cylindrical-ray test with no penumbra.
  * `sig_*_km` at each sample
  * `sgp4_err`: a nonzero value means that row is invalid
* `topo_bellevue_10s_72h.parquet`: 3,098,121 rows, only samples with the object above 0° elevation. Columns:
  * `az_deg` (from N through E), `el_deg` (geometric, no refraction)
  * `range_km`, `range_rate_kms` (positive means receding)
  * `sunlit`, `sun_el_deg` (Sun elevation at the observer)
  * `sig_along_km`, `sig_cross_km`
  * `sig_ang_deg` = atan-free approximation of sqrt(σa²+σc²)/range, an angular uncertainty upper bound
* `passes_bellevue_72h.csv`: 19,372 horizon passes.
  * Columns: rise, culmination and set time and azimuth, max elevation, min range, sunlit at culmination, `sunlit_and_dark_sky_any` (satellite sunlit while the Sun is below −6° at Bellevue), angular σ, tier and `truncated` (pass is cut off by the start or end of the window).
  * 12,648 passes have max elevation above 10°.
  * Geostationary objects show up as one long "pass."
* `calib_*`, `empirical_*`: validation data. `raw/`: everything downloaded (GP/SupGP JSON, McCants zips and TLEs, SeeSat-L HTML and parsed TLEs) plus `globe3d_before_hifi.html`.
* `scripts/`: run in order from `hifi/`: `build_elements.py`, `empirical.py`, `calib.py`, `gen_ephem.py`, `post.py`, `build_globe.py`, then `test_globe_hifi.py`. `seesat.py` parses the archive.
* Frame check: TEME→GCRS and ITRS positions match skyfield's `EarthSatellite` to <0.2 m and velocities to <1e-6 km/s. That checks only numerical consistency, not accuracy.

## 5. `../globe3d.html` (upgraded)
* Uses the selected freshest elements for all 1,069 objects.
* Colour by data quality tier (default) or by country.
* Clicking a satellite draws a 3σ RIC uncertainty ellipsoid that grows with simulated time. The ellipsoid is exaggerated when it would otherwise be too small to see, and the panel says when it is. The panel also shows σ along/cross/radial and the timing σ at sim time.
* Bellevue is shown with a marker, a label and optional lines of sight to every displayed satellite above the local horizon.
* A pass table lists max elevation ≥20°, excluding unreliable and truncated passes: 6,948 rows. Clicking a row jumps the clock to culmination and selects the satellite.

## 6. Limits
* **SGP4 and GP/TLE accuracy is the floor.** It is roughly a few hundred metres to a few km at epoch, growing by km/day. Nothing in this directory is meter-level, including the 60 s ephemeris with its sub-metre *numerical* precision. Treat the decimal places as numerical precision only.
* SupGP is better (about 0.1–0.5 km), but it covers only 25 objects in this set. These are mostly GLONASS, with 2 laser-ranging targets.
* **Classified objects (the McCants set):** the elements come from visual and optical observations by a small number of amateurs. Typical accuracy is km-level when fresh (a few seconds of timing in LEO). Errors drift by tens to hundreds of km after weeks. Objects that maneuvered since their last observation can be anywhere along the orbit, or in a different orbit. Many sets are months or years old (tiers D/E). No public source provides meter-level, or even sub-km, positions for these objects.
* No covariance, maneuvers, space weather or light-time is modelled. Light-time is ≤0.13 s even for GEO, which is far below the uncertainty. Elevation is geometric, with no refraction. The ephemerides should be regenerated as soon as new elements appear: rerun the scripts.
* Out of scope by design: Space-Track data, de-anonymising unpublished or temp-ID objects, and any targeting use. Telescope planning has not been built yet.
