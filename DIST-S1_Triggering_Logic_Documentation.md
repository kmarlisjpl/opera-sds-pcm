# DIST-S1 Algorithm Deployment: Triggering Logic and Workflow Documentation

## Table of Contents
1. [Executive Summary](#executive-summary)
2. [System Overview](#system-overview)
3. [Input Acquisition](#input-acquisition)
4. [Triggering Logic](#triggering-logic)
5. [Satellite Selection (S1A vs S1C)](#satellite-selection-s1a-vs-s1c)
6. [Date Selection and Acquisition Cycles](#date-selection-and-acquisition-cycles)
7. [Output Delivery](#output-delivery)
8. [Key Files Reference](#key-files-reference)

---

## Executive Summary

The DIST-S1 (Disturbance Alert for Sentinel-1) algorithm is an OPERA Level 3 product that detects land surface disturbances using Sentinel-1 SAR data. The system processes Radiometric Terrain Corrected (RTC) products organized by MGRS tiles and produces disturbance alert products.

**Key Characteristics:**
- **Input**: RTC-S1 products (Level 2) from Sentinel-1 satellites (S1A, S1C)
- **Output**: L3_DIST-ALERT-S1 products per MGRS tile
- **Temporal Resolution**: 12-day repeat cycle (per satellite)
- **Processing Mode**: Forward processing with baseline (historical) reference

---

## System Overview

### Architecture Components

The DIST-S1 deployment consists of several key components:

1. **Query System** (`rtc_for_dist_query.py`): Queries CMR for available RTC granules
2. **Burst Database** (`mgrs_burst_lookup_table.parquet`): Maps RTC bursts to MGRS tiles and products
3. **Triggering Engine** (`dist_s1_utils.py`): Determines when products are ready for processing
4. **Download System** (`asf_rtc_for_dist_download.py`): Manages RTC data retrieval
5. **Dependency Manager** (`dist_dependency.py`): Handles temporal dependencies between products
6. **PGE Wrapper** (`opera_pge_wrapper.py`): Executes the DIST-S1 algorithm

### Data Flow

```
CMR Query → RTC Granule Discovery → Burst Extension → Triggering Evaluation →
Download Job Submission → SCIFLO Job Submission → PGE Execution → Product Delivery
```

---

## Input Acquisition

### Burst Database

The DIST-S1 burst database is the foundation of the input acquisition system. It is a Parquet file (`mgrs_burst_lookup_table.parquet`) that defines:

- **MGRS tiles**: 30m MGRS grid cells (e.g., "33UVB")
- **Product IDs**: Unique acquisition groups within each tile (e.g., "33UVB_4")
- **Burst IDs**: Individual SAR burst identifiers (e.g., "T168-359429-IW2")

**Database Structure:**
```
mgrs_tile_id | acq_group_id_within_mgrs_tile | jpl_burst_id
-------------|------------------------------|-------------
33UVB        | 4                            | T168-359429-IW2
33UVB        | 4                            | T168-359430-IW2
...
```

**Key Functions:**
- `localize_dist_burst_db()`: Downloads and caches the burst database from S3
- `process_dist_burst_db()`: Parses the database into lookup structures:
  - `dist_products`: Maps tiles to product IDs
  - `bursts_to_products`: Maps burst IDs to product IDs (one-to-many)
  - `product_to_bursts`: Maps product IDs to required burst IDs
  - `all_tile_ids`: Complete list of MGRS tiles

**Performance Optimization:**
The database is pickled on first load for faster subsequent access (~15 MB pickle file vs. processing the full Parquet on each run).

### CMR Query Process

**Query Modes:**

1. **Forward Processing Mode** (Production):
   - Queries CMR for recent RTC granules based on time range
   - Populates/updates the CMR RTC cache index in ElasticSearch
   - Filters granules to only those in the burst database
   - Ensures cache gaps don't exceed 3 days (configurable: `MAX_CMR_RTC_CACHE_GAP_DAYS`)

2. **Reprocessing Mode**:
   - Triggered by specific product ID and acquisition time
   - Queries CMR for all bursts associated with the product
   - Example: `--product-id-time "31SGR_3,20231217T053132Z"`

**Query Parameters:**
- Collection: `OPERA_L2_RTC-S1_V1`
- Time Range: Configurable, typically last few hours for forward processing
- Native ID Pattern: Burst-specific RTC product identifiers

**CMR RTC Cache:**
The system maintains a cache (`cmr_rtc_cache` ES index) of all RTC granules to support dependency resolution:
- **Purpose**: Quickly determine previous products for baseline computation
- **Validation**: Minimum 30,000 documents, 20+ day range (configurable in `settings.yaml`)
- **Population**: Via `tools/populate_cmr_rtc_cache.py`

### Granule Decoration and Extension

After querying, granules are "decorated" with metadata and "extended" to handle multi-product bursts:

**Decoration (`basic_decorate_granule()`):**
```python
granule["burst_id"] = "T168-359429-IW2"
granule["acquisition_ts"] = datetime(2023, 12, 17, 5, 24, 15)
granule["acquisition_cycle"] = 302
granule["satellite"] = "S1A"  # or S1C
```

**Extension (`extend_rtc_for_dist_records()`):**
- Many RTC bursts belong to multiple DIST products
- Each granule is duplicated with different `product_id` assignments
- Example: Burst `T168-359429-IW2` might belong to products `33UVB_4` AND `33VVC_2`

**Unique Identification:**
```python
unique_id = f"{partial_granule_id}_{batch_id}"
# Example: "OPERA_L2_RTC-S1_T168-359429-IW2_20231217T052415Z_33UVB_4_S1A_302"
```

---

## Triggering Logic

### Overview

The triggering logic determines when a DIST-S1 product is ready to be processed. A product is triggered when:

1. **All required bursts are available** (complete burst set), OR
2. **Grace period has expired** (partial burst set allowed after timeout)

### Core Triggering Algorithm

**Function:** `compute_dist_s1_triggering()`
**Location:** `data_subscriber/dist_s1_utils.py:216`

**Inputs:**
- `product_to_bursts`: Mapping of product IDs to required burst IDs
- `denorm_granules_dict`: Dictionary of unique RTC granules (already extended)
- `complete_bursts_only`: Boolean flag for strict vs. grace-period mode
- `grace_mins`: Grace period in minutes (default: 210 minutes = 3.5 hours)
- `now`: Current datetime for grace period evaluation

**Algorithm Steps:**

1. **Accumulate Burst Coverage:**
   ```python
   for each granule in denorm_granules_dict:
       batch_id = derive_batch_id(granule)  # e.g., "33UVB_4_S1A_302"
       product_id = derive_product_id(batch_id)  # e.g., "33UVB_4"

       triggered_product[batch_id].used_bursts += 1
       triggered_product[batch_id].possible_bursts = len(product_to_bursts[product_id])
       triggered_product[batch_id].earliest_creation = min(creation_timestamps)
   ```

2. **Evaluate Completeness:**
   ```python
   if complete_bursts_only:
       for product in products_triggered:
           if product.used_bursts != product.possible_bursts:
               mins_since_creation = (now - product.earliest_creation).total_seconds() / 60

               if mins_since_creation < grace_mins:
                   # Remove from triggered list (wait for more data)
                   del products_triggered[product_id]
               else:
                   # Grace period expired, trigger with partial data
                   logger.info(f"Triggering {product_id} with {used}/{possible} bursts after {mins_since_creation} mins")
   ```

3. **Return Triggered Products:**
   - `products_triggered`: Dict of batch_id → DIST_S1_Product objects
   - `granules_triggered`: Dict of granule_id → boolean (used/unused)

### Batch ID Structure

**Batch ID Format:** `p{tile_id}_{acq_group}_S1{X}_a{acq_cycle}`

**Examples:**
- `p33UVB_4_S1A_a302`: Product for tile 33UVB, group 4, S1A satellite, cycle 302
- `p11SLT_1_S1C_a348`: Product for tile 11SLT, group 1, S1C satellite, cycle 348

**Components:**
- `p`: Prefix marker
- `tile_id`: MGRS tile identifier
- `acq_group`: Acquisition group within tile (multiple groups per tile)
- `satellite`: S1A or S1C (or future S1D)
- `acq_cycle`: Acquisition cycle index (more details in Date Selection section)

### Grace Period Behavior

**Configured in `settings.yaml`:**
```yaml
DIST_S1_TRIGGERING:
  DEFAULT_DIST_S1_QUERY_GRACE_PERIOD_MINUTES: 210
```

**Scenarios:**

1. **Complete Data Available:**
   - All bursts received within minutes of acquisition
   - Product triggers immediately (typical case)

2. **Partial Data, Within Grace Period:**
   - Some bursts missing (e.g., 8/10 bursts)
   - System waits for more data
   - Product NOT triggered yet

3. **Partial Data, Grace Period Expired:**
   - 210+ minutes since earliest burst creation
   - Product triggered with available bursts (e.g., 8/10)
   - Algorithm will adapt to missing data

**Timestamp Used:**
- Forward processing: CMR query timestamp
- Unsubmitted granules: `creation_timestamp` from ElasticSearch catalog

---

## Satellite Selection (S1A vs S1C)

### Satellite Mission Overview

**Active Satellites:**
- **Sentinel-1A (S1A)**: Launched April 2014, still operational
- **Sentinel-1C (S1C)**: Launched December 2023, replaced Sentinel-1B

**Retired:**
- **Sentinel-1B (S1B)**: Launched April 2016, decommissioned December 2021

**Future:**
- **Sentinel-1D (S1D)**: Planned launch (epoch TBD)

### Satellite-Specific Processing

**Key Principle:** Each satellite is processed independently. S1A and S1C products are NEVER mixed in a single DIST-S1 output.

**Implementation:**

1. **Granule Identification:**
   ```python
   # From granule filename parsing
   satellite = granule["granule_id"].split("_")[6]  # "S1A", "S1C", etc.
   ```

2. **Batch ID Segregation:**
   ```python
   batch_id = f"p{product_id}_{satellite}_a{acquisition_cycle}"
   # S1A product: "p33UVB_4_S1A_a302"
   # S1C product: "p33UVB_4_S1C_a348"
   ```

3. **Separate Acquisition Cycles:**
   - S1A and S1C have different epoch dates (6 days apart)
   - This creates distinct acquisition cycle indices
   - Even for the same geographic location, S1A and S1C products are separate

**Epoch Dates** (`rtc_utils.py`):
```python
_EPOCH_S1A = "20140101T000000Z"
_EPOCH_S1C = "20140107T000000Z"  # 6 days offset from S1A
```

### Why Satellites Don't Mix

**Architectural Guarantee:**

The batch ID includes the satellite identifier, ensuring:
- Triggering logic groups by `batch_id`
- Download jobs are satellite-specific
- Baseline retrieval filters by matching tile AND satellite
- PGE receives homogeneous satellite inputs

**Example:**
```
Tile 33UVB, Group 4, 2025-01-15:
- S1A bursts → batch "p33UVB_4_S1A_a350" → DIST product A
- S1C bursts → batch "p33UVB_4_S1C_a344" → DIST product B
(Two separate products, never combined)
```

### Polarization Handling

While not satellite-specific, polarization is similarly segregated:

**Polarization Preference:**
- **VV/VH (co-pol/cross-pol)**: Common for most regions
- **HH/HV**: Used in some regions (e.g., high latitudes)

**Mechanism:**
- `create_batch_id_to_polarizations_map()`: Determines dominant polarization per batch
- Single-polarization batches are rejected (need both co-pol and cross-pol)
- Mixed polarizations within a batch are allowed (algorithm adapts)

---

## Date Selection and Acquisition Cycles

### Acquisition Cycle Concept

**Definition:**
An acquisition cycle is an integer index representing the number of 12-day periods elapsed since a satellite's mission epoch.

**12-Day Repeat Cycle:**
- Sentinel-1 satellites revisit the same location every 12 days
- This creates a consistent temporal cadence for change detection
- Acquisition cycle index allows temporal alignment across years

### Acquisition Cycle Calculation

**Function:** `determine_acquisition_cycle()`
**Location:** `rtc_utils.py:73`

**Algorithm:**
```python
def determine_acquisition_cycle(burst_id, acquisition_dts, granule_id, epoch=None):
    cycle_days = 12
    satellite = granule_id.split("_")[6]  # S1A, S1B, S1C, S1D

    instrument_epoch = isoparse(_EPOCH_MAP[satellite])
    MAX_BURST_IDENTIFICATION_NUMBER = 375887
    ACQUISITION_CYCLE_DURATION_SECS = timedelta(days=12).total_seconds()

    burst_identification_number = int(burst_id.split("-")[1])
    seconds_after_mission_epoch = (isoparse(acquisition_dts) - instrument_epoch).total_seconds()

    acquisition_index = (
        seconds_after_mission_epoch -
        (ACQUISITION_CYCLE_DURATION_SECS * (burst_identification_number / MAX_BURST_IDENTIFICATION_NUMBER))
    ) / ACQUISITION_CYCLE_DURATION_SECS

    acquisition_cycle = round(acquisition_index)
    return acquisition_cycle
```

**Example Calculation:**

Given:
- Satellite: S1A
- Burst ID: T168-359429-IW2
- Acquisition: 2023-12-17T05:24:15Z

Steps:
1. Epoch: 2014-01-01T00:00:00Z
2. Seconds since epoch: ~312,000,000 seconds
3. Burst offset: (359429 / 375887) * 12 days ≈ 11.48 days
4. Adjusted seconds: ~311,008,000 seconds
5. Acquisition cycle: 311,008,000 / (12 * 86400) ≈ 302

**Result:** `acquisition_cycle = 302`

### Baseline (k-) Retrieval

**Purpose:**
DIST-S1 requires historical "baseline" RTC data for comparison against current acquisitions to detect changes.

**K-Parameter Configuration:**
```python
K_OFFSETS_AND_COUNTS = "[(365, 4), (730, 3), (1095, 3)]"
```

**Interpretation:**
- Look back 365 days (±12 days), retrieve 4 acquisitions
- Look back 730 days (±24 days), retrieve 3 acquisitions
- Look back 1095 days (±36 days), retrieve 3 acquisitions

**Function:** `retrieve_baseline_granules()`
**Location:** `data_subscriber/rtc_for_dist/rtc_for_dist_query.py:261`

**Algorithm:**

1. **Time Window Construction:**
   ```python
   for k_offset, k_count in k_offsets_and_counts:
       shift_day_grouping = 12 * (k_count * DIST_K_MULT_FACTOR)  # DIST_K_MULT_FACTOR = 2
       counter = 1

       while k_satisfied < k_count:
           start_date_shift = timedelta(days=k_offset + counter * shift_day_grouping, hours=1)
           end_date_shift = timedelta(days=k_offset + (counter-1) * shift_day_grouping, hours=1)

           start_date = (acquisition_time - start_date_shift).strftime(CMR_TIME_FORMAT)
           end_date = (acquisition_time - end_date_shift).strftime(CMR_TIME_FORMAT)
   ```

2. **CMR Query:**
   - Query for RTC granules in time window
   - Filter by native_id (matching burst IDs only)

3. **Acquisition Cycle Sorting:**
   - Group baseline granules by acquisition cycle
   - Sort in descending order (most recent first)
   - Select top `k_count` cycles

4. **Per-Burst Limiting:**
   ```python
   for granule in possible_k_granules:
       burst_id = extract_burst_id(granule)
       if len(burst_id_to_granules_map[burst_id]) >= k_count:
           continue  # Skip extra granules per burst
       burst_id_to_granules_map[burst_id].append(granule)
   ```

**Example:**
For `acquisition_time = 2023-12-17`, offset 365 days, count 4:
- Window 1: 2022-12-17 ± 24 days
- Retrieve acquisitions with cycles: ~272, ~273, ~274, ~275
- Result: 4 historical acquisitions ~1 year prior

### High Latitude Considerations

**Question:** Does DIST-S1 handle high latitudes differently?

**Answer:** No explicit high-latitude logic, but indirect effects:

1. **Acquisition Cycle Calculation:**
   - Formula includes `burst_identification_number` offset
   - Different bursts have different cycle phase alignments
   - High-latitude bursts may have different IDs, thus different phases

2. **Polarization:**
   - High latitudes often use HH/HV polarization
   - System adapts by filtering baseline data to match current polarization

3. **Coverage Density:**
   - High latitudes have more frequent coverage (orbit convergence)
   - Burst database reflects actual coverage patterns
   - Products may be smaller (fewer bursts per tile)

4. **Date Arithmetic:**
   - 12-day cycle is consistent globally
   - No seasonal or latitude-based adjustments

**No Special Date Selection:**
The date selection logic (`determine_acquisition_cycle()`) is purely temporal and geometric (burst ID offset), with no latitude-dependent conditionals.

---

## Output Delivery

### Download Job Submission

**Function:** `download_job_submission_handler()`
**Location:** `data_subscriber/rtc_for_dist/rtc_for_dist_query.py:346`

**Steps:**

1. **Polarization Analysis:**
   - Determine dominant polarization for current granules
   - Filter baseline and current RTC URLs by polarization preference
   - Reject single-polarization batches

2. **S3 Path Collection:**
   - `current_s3_paths`: Current acquisition RTC S3 URLs
   - `baseline_s3_paths`: Baseline (k-) RTC S3 URLs
   - `previous_tile_product_file_paths`: Previous DIST product (if exists)

3. **Dependency Check:**
   - Call `dist_dependency.should_wait_previous_run()`
   - Determine if previous tile product is available
   - If missing and job is pending: create pending download job
   - If available: proceed with download job

4. **Product Metadata Construction:**
   ```python
   product_metadata = {
       "current_s3_paths": sorted(current_urls),
       "baseline_s3_paths": sorted(baseline_urls),
       "previous_tile_product_file_paths": previous_paths  # or None
   }
   ```

5. **Job Submission:**
   - Job type: `rtc_for_dist_download`
   - Job name: `job-WF-rtc_for_dist_download-{batch_id}`
   - Queue: `self.args.job_queue`
   - Params: Include `product_metadata` object

### RTC Download Execution

**Function:** `run_download()`
**Location:** `data_subscriber/asf_rtc_for_dist_download.py:30`

**Process:**

1. **Load Job Context:**
   - Read `_context.json` from job working directory
   - Extract `product_metadata`

2. **S3 Direct Access (Preferred):**
   - RTC files remain in ASF S3 bucket
   - No local download (saves time and space)
   - S3 paths passed directly to SCIFLO job

3. **HTTPS Fallback (If Configured):**
   - Download RTC files to local worker
   - Upload to OPERA staging bucket
   - Use OPERA S3 paths for SCIFLO

4. **SCIFLO Job Submission:**
   ```python
   product = {
       "_id": batch_id,
       "_source": {
           "dataset": f"L3_DIST_S1-{batch_id}",
           "metadata": {
               "batch_id": batch_id,
               "mgrs_tile_id": tile_id,
               "product_paths": {
                   "L2_RTC_S1": {
                       "baseline_burst_set": baseline_s3paths,
                       "current_burst_set": current_s3paths,
                   },
                   "L3_DIST_S1": previous_tile_product_file_paths,
               },
               "acquisition_cycle": acquisition_cycle_index,
               "satellite": satellite,
               ...
           }
       }
   }

   try_submit_mozart_job(
       product=product,
       job_queue='opera-job_worker-sciflo-l3_dist_s1',
       job_spec=f'job-SCIFLO_L3_DIST_S1:{RELEASE_VERSION}',
       job_name=f'job-WF-SCIFLO_L3_DIST_S1-batch-{batch_id}',
       ...
   )
   ```

5. **Catalog Update:**
   - Mark RTC granules as downloaded in ElasticSearch
   - Record download job ID
   - Associate granules with SCIFLO job

### SCIFLO PGE Execution

**Workflow:** `opera_chimera/wf_xml/L3_DIST_S1.sf.xml`

**PGE Configuration:** `opera_chimera/configs/pge_configs/PGE_L3_DIST_S1.yaml`

**Precondition Functions:**

1. `get_product_version`: Retrieve DIST-S1 version from settings
2. `get_cnm_version`: Determine CNM message version
3. `set_daac_product_type`: Set collection name for DAAC delivery
4. `get_dist_s1_rtc_s3_paths`: Organize RTC inputs into pre/post, co-pol/cross-pol lists
5. `get_dist_s1_prev_product`: Locate previous DIST product directory
6. `get_dist_s1_mgrs_tile`: Extract MGRS tile ID
7. `get_dist_s1_mask_file`: Stage water mask for tile
8. `get_static_ancillary_files`: Download algorithm parameters YAML

**Function:** `get_dist_s1_rtc_s3_paths()`
**Location:** `opera_chimera/precondition_functions.py:804`

**Sorting Logic:**
```python
rtc_pattern = re.compile(r'..._(?P<burst_id>\w{4}-\w{6}-\w{3})_.*_(?P<acquisition_ts>\d{8}T\d{6}Z)_.*')

for path in product_paths["baseline_burst_set"]:
    if polarization in ['VV', 'HH']:
        pre_copol.append(path)
    else:
        pre_crosspol.append(path)

for path in product_paths["current_burst_set"]:
    if polarization in ['VV', 'HH']:
        post_copol.append(path)
    else:
        post_crosspol.append(path)

# Sort by (burst_id, acquisition_ts)
pre_copol.sort(key=lambda path: (extract_burst_id(path), extract_acq_ts(path)))
...
```

**RunConfig Template:** `conf/RunConfig.yaml.L3_DIST_S1.jinja2.tmpl`

**Key Parameters:**
```yaml
RunConfig:
  Groups:
    PGE:
      InputFilesGroup:
        InputFilePaths:
          - {pre_rtc_copol}      # Baseline co-pol
          - {pre_rtc_crosspol}   # Baseline cross-pol
          - {post_rtc_copol}     # Current co-pol
          - {post_rtc_crosspol}  # Current cross-pol
          - {prev_product}       # Previous DIST product (if exists)
      DynamicAncillaryFilesGroup:
        AncillaryFileMap:
          src_water_mask_path: {src_water_mask_path}
      PrimaryExecutable:
        ProgramPath: /opt/conda/envs/dist-s1-env/bin/dist-s1
        ProgramOptions:
          - run_sas
          - --run_config_path
    SAS:
      run_config:
        pre_rtc_copol: [...]
        pre_rtc_crosspol: [...]
        post_rtc_copol: [...]
        post_rtc_crosspol: [...]
        prior_dist_s1_product: {...}
        mgrs_tile_id: {mgrs_tile_id}
        algo_config_path: {algorithm_parameters_file}
```

### Product Output

**Output Files:**
- `OPERA_L3_DIST-ALERT-S1_{tile}_{acq_ts}Z_{creation_ts}Z_{sat}_30_v{ver}_*.tif`
- `*.png` (browse images)
- `*.catalog.json` (metadata)
- `*.iso.xml` (ISO metadata)
- `*.log` (processing logs)

**Metadata Fields:**
- MGRS tile ID
- Acquisition timestamp (current acquisition)
- Creation timestamp
- Satellite (S1A or S1C)
- Product version
- Bounding box
- Input RTC granule IDs
- Baseline granule IDs
- Algorithm parameters

**Delivery:**
1. **Staging:** Products written to S3 staging area
2. **Cataloging:** Indexed in GRQ ElasticSearch
3. **CNM Notification:** DAAC notified via SNS
4. **DAAC Delivery:** Products transferred to DAAC (ASF)

---

## Key Files Reference

### Core Logic Files

| File | Purpose |
|------|---------|
| `data_subscriber/dist_s1_utils.py` | Burst database processing, triggering logic |
| `data_subscriber/rtc_for_dist/rtc_for_dist_query.py` | CMR querying, baseline retrieval, download job submission |
| `data_subscriber/rtc_for_dist/dist_dependency.py` | Previous product dependency management |
| `data_subscriber/asf_rtc_for_dist_download.py` | RTC download and SCIFLO job submission |
| `rtc_utils.py` | Acquisition cycle calculation, satellite epochs |

### Configuration Files

| File | Purpose |
|------|---------|
| `conf/settings.yaml` | System-wide settings (grace period, cache thresholds, S3 paths) |
| `opera_chimera/configs/pge_configs/PGE_L3_DIST_S1.yaml` | PGE execution configuration |
| `conf/RunConfig.yaml.L3_DIST_S1.jinja2.tmpl` | PGE RunConfig template |
| `conf/schema/AlgoParams_schema.L3_DIST_S1.yaml` | Algorithm parameter schema |

### Job Definitions

| File | Purpose |
|------|---------|
| `docker/hysds-io.json.SCIFLO_L3_DIST_S1` | SCIFLO job specification |
| `docker/job-spec.json.SCIFLO_L3_DIST_S1` | Job execution parameters |
| `docker/hysds-io.json.rtc_for_dist_download` | Download job specification |

### Tools

| File | Purpose |
|------|---------|
| `tools/dist_s1_burst_db_tool.py` | Query and analyze burst database |
| `tools/populate_cmr_rtc_cache.py` | Populate RTC cache in ElasticSearch |
| `data_subscriber/submit_pending_jobs.py` | Process pending download jobs |

### Workflow

| File | Purpose |
|------|---------|
| `opera_chimera/wf_xml/L3_DIST_S1.sf.xml` | SCIFLO workflow definition |
| `opera_chimera/precondition_functions.py` | Pre-execution setup (lines 788-929) |
| `wrapper/opera_pge_wrapper.py` | PGE wrapper execution |

---

## Appendix: Triggering Logic Detailed Example

### Scenario

**Tile:** 33UVB
**Product:** 33UVB_4
**Satellite:** S1A
**Acquisition Cycle:** 302
**Date:** 2023-12-17

**Required Bursts (from database):**
1. T168-359429-IW2
2. T168-359430-IW2
3. T168-359431-IW2
4. T169-359724-IW1
5. T169-359725-IW1

### Timeline

**T+0 (05:24:15 UTC):**
- RTC granule 1 arrives in CMR: `OPERA_L2_RTC-S1_T168-359429-IW2_20231217T052415Z_..._S1A_30_v1.0`

**T+2 mins:**
- RTC granule 2 arrives: `OPERA_L2_RTC-S1_T168-359430-IW2_20231217T052417Z_..._S1A_30_v1.0`

**T+5 mins:**
- RTC granule 3 arrives: `OPERA_L2_RTC-S1_T168-359431-IW2_20231217T052419Z_..._S1A_30_v1.0`

**T+7 mins:**
- RTC granule 4 arrives: `OPERA_L2_RTC-S1_T169-359724-IW1_20231217T052422Z_..._S1A_30_v1.0`

**T+10 mins:**
- Query job runs
- Granules 1-4 are in ElasticSearch catalog
- Triggering evaluation:
  - Product 33UVB_4: 4/5 bursts available
  - Grace period: 10 mins < 210 mins
  - **Decision:** Wait for more data
  - No download job submitted

**T+4 hours:**
- No additional granules arrived
- Query job runs again
- Triggering evaluation:
  - Product 33UVB_4: still 4/5 bursts
  - Grace period: 240 mins > 210 mins
  - **Decision:** Grace period expired, trigger with partial data
  - Download job submitted

**Download Job:**
- Batch ID: `p33UVB_4_S1A_a302`
- Current burst set: 4 RTC granules
- Baseline retrieval: Query CMR for cycles 301, 290, 278, 266, 254, 242, ...
- Previous product lookup: Search GRQ for `33UVB_4_S1A_a301` output

**SCIFLO Job:**
- Input: 4 current RTCs + ~40 baseline RTCs + previous DIST product
- Algorithm: DIST-S1 SAS processes all inputs
- Output: DIST-ALERT-S1 product for 33UVB tile

---

## Summary

The DIST-S1 triggering logic is a sophisticated orchestration system that:

1. **Discovers** RTC products via CMR queries filtered by a burst database
2. **Triggers** processing when burst sets are complete or grace period expires
3. **Segregates** S1A and S1C data streams into separate products
4. **Selects** baseline data using acquisition cycle arithmetic and k-parameter windows
5. **Delivers** products through a staged download → SCIFLO → PGE → DAAC pipeline

The system is designed for robustness (grace periods, partial data handling), efficiency (pickle caching, S3 direct access), and correctness (satellite segregation, polarization matching, dependency tracking).
