# S1A vs S1C Segregation: Detailed Code Analysis

## Question
How does the system ensure S1A and S1C data don't get mixed together? Is it the 6-day offset or something more explicit?

## Answer
It's **BOTH**, but with an important finding: **baseline granules are NOT explicitly filtered by satellite**.

---

## Mechanism 1: Explicit Satellite Embedding in Batch IDs

### Current Granules (EXPLICIT SEGREGATION ✅)

**Step 1: Satellite Extraction**
```python
# data_subscriber/dist_s1_utils.py:162
granule["satellite"] = granule["granule_id"].split("_")[6]  # "S1A", "S1C", etc.
```
From filename: `OPERA_L2_RTC-S1_T168-359429-IW2_20231217T052415Z_20231220T055805Z_S1A_30_v1.0`
                                                                                    ^^^^
                                                                                    Position 6

**Step 2: Batch ID Construction**
```python
# data_subscriber/dist_s1_utils.py:166
granule["batch_id"] = granule["product_id"] + "_" + granule["satellite"] + "_" + str(granule["acquisition_cycle"])
```
Example: `33UVB_4` + `_S1A_` + `302` = `33UVB_4_S1A_302`

**Step 3: Download Batch ID**
```python
# data_subscriber/dist_s1_utils.py:124
download_batch_id = "p" + str(granule["product_id"]) + "_" + str(granule["satellite"]) + "_a" + str(granule["acquisition_cycle"])
```
Example: `p33UVB_4_S1A_a302`

**Result:** Current granules are **explicitly segregated by satellite**.

---

## Mechanism 2: Acquisition Cycle Calculation (IMPLICIT SEGREGATION via 6-day offset)

### Different Epochs Create Different Cycles

```python
# rtc_utils.py:7-16
_EPOCH_S1A = "20140101T000000Z"
_EPOCH_S1C = "20140107T000000Z"  # 6 days offset

def determine_acquisition_cycle(burst_id, acquisition_dts, granule_id, epoch=None):
    satellite = granule_id.split("_")[6]
    instrument_epoch = isoparse(_EPOCH_MAP[satellite])  # Different epoch per satellite!

    acquisition_cycle = round(
        (isoparse(acquisition_dts) - instrument_epoch).total_seconds() / (12 * 86400)
    )
```

**Example for same acquisition time:**
- S1A burst acquired 2023-12-17: cycle ≈ 302
- S1C burst acquired 2023-12-17: cycle ≈ 296 (6 days / 12 days ≈ 0.5 cycle offset)

**Result:** Even for the same location/time, S1A and S1C get **different acquisition cycle indices**.

---

## Mechanism 3: Triggering Groups by Batch ID (EXPLICIT SEGREGATION ✅)

```python
# data_subscriber/dist_s1_utils.py:216
def compute_dist_s1_triggering(product_to_bursts, denorm_granules_dict, ...):
    for d_g, granule in denorm_granules_dict.items():
        batch_id = d_g[1]  # e.g., "33UVB_4_S1A_302"
        products_triggered[batch_id] = ...  # Keyed by batch_id (includes satellite!)
```

**Result:** Triggering creates **separate products** for S1A vs S1C.

---

## THE CRITICAL FINDING: Baseline Granules Are NOT Filtered by Satellite! ⚠️

### Baseline Retrieval Analysis

**Step 1: Baseline Retrieval Request** (`rtc_for_dist_query.py:251-252`)
```python
download_batch_id = batch_granules[0]["download_batch_id"]  # e.g., "p33UVB_4_S1A_a302"
product_id = "_".join(batch_id.split("_")[0:2])              # e.g., "33UVB_4" (NO satellite!)

self.batch_id_to_k_granules[download_batch_id] = (
    self.retrieve_baseline_granules(product_id, batch_granules, ...)  # Uses product_id without satellite!
)
```

**Step 2: CMR Query for Baseline** (`rtc_for_dist_query.py:275`)
```python
_, new_args.native_id = build_rtc_native_ids(product_id, self.product_to_bursts)
# Queries CMR for ALL RTC granules matching these burst IDs, regardless of satellite!
```

**Step 3: Baseline Granule Decoration** (`rtc_for_dist_query.py:308-310`)
```python
for granule in granules:  # These are baseline granules from CMR
    basic_decorate_granule(granule)  # Extracts satellite from EACH granule's filename
    granule["product_id"] = product_id  # Forces same product_id (no satellite)
```

**Step 4: Baseline URL Collection** (`rtc_for_dist_query.py:444-451`)
```python
batch_id_to_baseline_urls = defaultdict(list)
for download_batch_id, granules in self.batch_id_to_k_granules.items():
    # download_batch_id = "p33UVB_4_S1A_a302" (from current granules)
    for granule in granules:  # Baseline granules
        # Each baseline granule has its own satellite in granule["download_batch_id"]!
        # But all URLs go into batch_id_to_baseline_urls[download_batch_id]
        add_filtered_urls(granule, batch_id_to_baseline_urls[download_batch_id], ...)
```

### What This Means

**For a current S1A product:**
- Current granules: All S1A (batch_id = `p33UVB_4_S1A_a302`)
- Baseline query: Requests bursts for `product_id = "33UVB_4"` (no satellite specified)
- Baseline results from CMR: **Could include both S1A and S1C granules!**

**Example scenario:**
```
Current product: p33UVB_4_S1A_a302 (acquired 2023-12-17)

Baseline query for 1 year ago (2022-12-17):
  ├─ S1A granules from cycle ~271 ✓ Included
  ├─ S1C granules from cycle ~265 ✓ Also included! (if S1C was operational)
  └─ All go into the same baseline_s3_paths list!
```

### Why This (Probably) Works in Practice

1. **S1C is new (Dec 2023):** For most historical data, only S1A exists
2. **Acquisition cycle differences:** S1A cycle 271 and S1C cycle 265 are at different times
3. **Temporal filtering may separate them:** The baseline retrieval windows might naturally exclude the other satellite

However, there's **no explicit filtering** to prevent mixing!

---

## Verification Needed

To confirm whether baseline granules can mix satellites, check:

1. **Does the DIST-S1 algorithm handle mixed-satellite baselines?**
   - If yes: System is correct as-is
   - If no: Need to add satellite filtering in baseline retrieval

2. **Suggested fix location:** `rtc_for_dist_query.py:308-310`
   ```python
   # Current code:
   for granule in granules:
       basic_decorate_granule(granule)
       granule["product_id"] = product_id

   # Potential fix:
   current_satellite = downloads[0]["satellite"]  # Get satellite from current granules
   for granule in granules:
       basic_decorate_granule(granule)
       if granule["satellite"] != current_satellite:
           continue  # Skip baseline granules from different satellites
       granule["product_id"] = product_id
   ```

---

## Summary Table

| Mechanism | Current Granules | Baseline Granules |
|-----------|------------------|-------------------|
| **Satellite extraction from filename** | ✅ Yes (line 162) | ✅ Yes (line 309) |
| **Batch ID includes satellite** | ✅ Yes (line 166) | ✅ Yes (line 167 after decoration) |
| **Triggering segregated by satellite** | ✅ Yes (keyed by batch_id) | N/A |
| **CMR query filters by satellite** | ✅ Yes (implicitly via granule_id) | ❌ **NO** (queries by burst_id only) |
| **Explicit satellite filtering** | ✅ Yes | ❌ **NO** |

---

## Conclusion

**Current granules:** Explicitly segregated by satellite through batch_id mechanism.

**Baseline granules:** **NOT explicitly filtered by satellite!** They could theoretically mix S1A and S1C, though in practice:
- Temporal separation (6-day offset → different acquisition cycles)
- S1C is new (Dec 2023), so limited historical overlap
- Algorithm may handle mixed satellites gracefully

**Recommendation:** Add explicit satellite filtering in baseline retrieval to guarantee segregation.
