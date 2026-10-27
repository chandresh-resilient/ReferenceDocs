# Test Plan & Verification Specification: Battery Measurements Ingestion System

A comprehensive, industry-grade test specification for verifying multi-vendor BMS log imports, measuring instrument files, multi-file upload lifecycle transitions, domain model synchronization, and UI data presentation via `Battery.SNIP_ViewOrEdit_Measures` in Mendix.

---

## Table of Contents

- [1. Executive Test Strategy & Governance](#1-executive-test-strategy--governance)
  - [Scope & Verification Objectives](#scope--verification-objectives)
  - [Test Execution Environments](#test-execution-environments)
  - [Preconditions, Test Isolation & Data Rollback](#preconditions-test-isolation--data-rollback)
  - [Standard Test Case Specification Template](#standard-test-case-specification-template)
- [2. Single File Upload Test Cases by File Type & Hardware Format](#2-single-file-upload-test-cases-by-file-type--hardware-format)
  - [Type 1: Samsung SDI BATTMON (`SAMSUNG_BATTMON`)](#type-1-samsung-sdi-battmon-samsung_battmon)
  - [Type 2: Samsung SDI ELPMON (`SAMSUNG_ELPMON`)](#type-2-samsung-sdi-elpmon-samsung_elpmon)
  - [Type 3: Samsung System Summary Log Guard (`SAMSUNG_SYSTEM_LOG`)](#type-3-samsung-system-summary-log-guard-samsung_system_log)
  - [Type 4: HPL / Vertiv VEC Lithium Snapshots (`HPL_VEC_SNAPSHOT`)](#type-4-hpl--vertiv-vec-lithium-snapshots-hpl_vec_snapshot)
  - [Type 5: Lithium Pack BMS Multi-Pack Logs (`LITHIUM_PACK_BMS`)](#type-5-lithium-pack-bms-multi-pack-logs-lithium_pack_bms)
  - [Type 6: Alber BDSU Backup Archives & Databases (`ALBER_BDSU`)](#type-6-alber-bdsu-backup-archives--databases-alber_bdsu)
  - [Type 7: Alber Cellcorder / CDF (`ALBER_CDF`)](#type-7-alber-cellcorder--cdf-alber_cdf)
  - [Type 8: Hioki BT3554 / BT3554-50 Battery HiTester (`HIOKI_BT3554`)](#type-8-hioki-bt3554--bt3554-50-battery-hitester-hioki_bt3554)
  - [Type 9: PowerShield Continuous Monitor (`HIOKI_BT3554` Mapped)](#type-9-powershield-continuous-monitor-hioki_bt3554-mapped)
  - [Type 10: Hioki 3554 Legacy HiTester (`HIOKI_3554_LEGACY`)](#type-10-hioki-3554-legacy-hitester-hioki_3554_legacy)
  - [Type 11: Fluke BT500 / BT510 / BT521 Analyzer (`FLUKE`)](#type-11-fluke-bt500--bt510--bt521-analyzer-fluke)
  - [Type 12: ZincFive Nickel-Zinc BMS Logs (`ZINC`)](#type-12-zincfive-nickel-zinc-bms-logs-zinc)
  - [Type 13: Generic Tabular Fallback (`GENERIC_TABULAR`)](#type-13-generic-tabular-fallback-generic_tabular)
- [3. Module Upload Mode (`isModuleUpload = true`) Test Cases](#3-module-upload-mode-ismoduleupload--true-test-cases)
- [4. Multiple File Upload & Re-Upload Scenarios (Lifecycle & Cross-Format Transitions)](#4-multiple-file-upload--re-upload-scenarios-lifecycle--cross-format-transitions)
  - [Scenario A: Alber CDF Upload Followed by Module-Only Hioki Re-Upload](#scenario-a-alber-cdf-upload-followed-by-module-only-hioki-re-upload)
  - [Scenario B: Module-Only Hioki Upload Followed by Full Alber CDF Re-Upload](#scenario-b-module-only-hioki-upload-followed-by-full-alber-cdf-re-upload)
  - [Scenario C: Lithium BMS Log Upload Followed by Portable Hioki Module-Level Upload](#scenario-c-lithium-bms-log-upload-followed-by-portable-hioki-module-level-upload)
  - [Scenario D: Multi-Rack Sequential Uploads (Rack 1 then Rack 2)](#scenario-d-multi-rack-sequential-uploads-rack-1-then-rack-2)
  - [Scenario E: Consecutive Upload of Exact Same File (Idempotency)](#scenario-e-consecutive-upload-of-exact-same-file-idempotency)
  - [Scenario F: Format Cross-Validation Mismatch Guard](#scenario-f-format-cross-validation-mismatch-guard)
  - [Scenario G: Battery Geometry Reconfiguration Mid-Job](#scenario-g-battery-geometry-reconfiguration-mid-job)
  - [Scenario H: Corrupted / Partial File Upload Rollback](#scenario-h-corrupted--partial-file-upload-rollback)
- [5. Edge Cases, Boundary Conditions & System Faults](#5-edge-cases-boundary-conditions--system-faults)
- [6. UI Verification & `SNIP_ViewOrEdit_Measures` Test Cases](#6-ui-verification--snip_vieworedit_measures-test-cases)
  - [DataGrid Rendering & Bulk Ingestion (`DS_GetBatteryMeasuresAdvanced`)](#datagrid-rendering--bulk-ingestion-ds_getbatterymeasuresadvanced)
  - [DOM Injected Grouped Headers (`Incentro JavaScriptSnippet`)](#dom-injected-grouped-headers-incentro-javascriptsnippet)
  - [Sticky Frozen Columns Behavior & CSS Engine](#sticky-frozen-columns-behavior--css-engine)
  - [In-Line Cell Editing & Dynamic Variance Commit Flow](#in-line-cell-editing--dynamic-variance-commit-flow)
  - [Client-Side Chart Generation (`getMeasuresChartJSON.js`)](#client-side-chart-generation-getmeasureschartjsonjs)
  - [Server-Side Java Chart Generation (`JA_GetMeasuresChartJSON`)](#server-side-java-chart-generation-ja_getmeasureschartjson)
- [7. Traceability Matrix & Execution Checklist](#7-traceability-matrix--execution-checklist)

---

## 1. Executive Test Strategy & Governance

### Scope & Verification Objectives
This specification governs end-to-end quality assurance for the Battery Measurements Ingestion System. The system spans client-side Nanoflow orchestration (`Battery.ACT_Upload_BatteryMeasures`), pure JavaScript extraction engines (`FileUploadJSA.JSA_ImportBMSData`), domain model synchronization (`Battery.BatteryMeasure`), and interactive UI presentation via `Battery.SNIP_ViewOrEdit_Measures`.

```
┌──────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                  END-TO-END TEST ARCHITECTURE                                    │
│                                                                                                  │
│   [ Client Browser / Mobile Container ]                                                          │
│   ├── UI Layer: Battery.SNIP_UploadBatteryMeasures ──► User selects file                        │
│   ├── Nanoflow Orchestrator: ACT_Upload_BatteryMeasures                                          │
│   │   ├── SUB_Battery_ResolveUploadContext (MIME, FileUploadType, Jars, Points)                  │
│   │   ├── SUB_Battery_CalculateExpectedCellCount (Math: jars x cells vs. module-only)           │
│   │   ├── SUB_Battery_ParseAndValidateBMS ──► Invokes JSA_ImportBMSData                          │
│   │   └── SUB_Battery_SyncBatteryMeasures (In-Memory Diff & Atomic Database Sync)                │
│   ├── Pure JS Engine: FileUploadJSA (Parsers, BLOB decoders, Geometry regrouping)                │
│   └── UI Presentation Layer: Battery.SNIP_ViewOrEdit_Measures                                    │
│       ├── Incentro JavaScriptSnippet: Grouped headers, sticky frozen columns, observers          │
│       ├── DataGrid: DS_GetBatteryMeasuresAdvanced, in-line editing, OCH_CommitBatteryMeasures    │
│       └── Visualization: getMeasuresChartJSON.js (Client) / JA_GetMeasuresChartJSON (Server)    │
└──────────────────────────────────────────────────────────────────────────────────────────────────┘
```

### Test Execution Environments
1. **Desktop Web Browsers**: Latest versions of Google Chrome, Microsoft Edge, and Mozilla Firefox (Windows 10/11 & macOS).
2. **Mobile Offline PWAs / Hybrid Containers**: Mendix Offline Native / PWA running client-side SQLite and IndexedDB.
3. **Headless Regression Harness**: Node.js automated test runner executing pure JavaScript parsers against sample data files (`test_bms_harness.html` and Node test suites).

### Preconditions, Test Isolation & Data Rollback
- Every test execution begins with a freshly initialized `Core.Job` and `DynamicQuestionnaire.Questionnaire`.
- In case of upload cancellation or validation failure, `Battery.SUB_Battery_CleanupFailedUpload` must execute atomically:
  1. Purges the uploaded file from `DynamicQuestionnaire.Media`.
  2. Executes `Battery.Sub_deleteAllMeasurementPOints` to remove any partially created `Battery.BatteryMeasure` records.
- Tests verify that failed imports leave the database in its exact pre-upload state without orphaned entities.

### Standard Test Case Specification Template
Each test case defines:
* **Scenario ID & Title**: Unique identifier and descriptive name.
* **Objective**: What requirement or bug fix is under verification.
* **Preconditions**: Initial questionnaire tags, string configuration, and existing database state.
* **Input File & Parameters**: File payload, `jarsPerBattery`, `measurementPoints`, `rack`, `isModuleUpload`, `shouldValidateCellCount`.
* **Expected Output (`BatteryData`)**: Detected `BatteryType`, module count, cell count per module, error string.
* **Expected Output (`Battery.BatteryMeasure`)**: Count of persistent records, attribute mappings (`val_bat_measure_point_float_voltage`, `val_bat_measure_point_resistance`, `val_bat_measure_point_impedance`, `val_bat_measure_point_tempc`, `val_bat_block_data_table`, `val_bat_measure_point_no_data_table`).
* **Pass/Fail Criteria**: Explicit boolean checks and assertion gates.

---

## 2. Single File Upload Test Cases by File Type & Hardware Format

### Type 1: Samsung SDI BATTMON (`SAMSUNG_BATTMON`)

#### TC-BAT-01: Samsung SDI BATTMON Standard 128S Configuration
* **Scenario ID**: `TC-BAT-01`
* **Objective**: Verify standard 16-module Samsung BATTMON CSV import with paired temperature sensor mapping ($1 \text{ sensor} / 2 \text{ cells}$) and latest row extraction.
* **Preconditions**:
  - `batterytype` = `'Lithium'`
  - `st_select_file_type_for_import` = `'BMS log file'`
  - `dc_source_bat_string_jars_blocs_per_string` = `16`
  - `dc_source_bat_string_cells_per_bloc` = `'8'`
  - `measurementPoints` = `128` ($16 \times 8$)
* **Input File**: `RACK001_Log_CELL_0.csv` (Contains 100 rows, columns `[RACK#001]`, `M#01_Cell#01[V]` .. `M#16_Cell#08[V]`, and `Temp#01` .. `Temp#04` per module).
* **Execution Parameters**:
  - `jarsPerBattery` = `16`
  - `measurementPoints` = `128`
  - `rack` = `"1"`
  - `isModuleUpload` = `false`
  - `shouldValidateCellCount` = `false`
* **Expected Result**:
  1. `BatteryData.BatteryType` = `Battery.Enum_BatteryType.SAMSUNG_BATTMON`.
  2. `BatteryData.Rack` = `"1"`, `BatteryData.RackNumber` = `1`.
  3. Total parsed `Module` entities = `16`.
  4. Each `Module` contains exactly `8` `CellInfo` entities (`CellIndex` $1 \dots 8$, `ModelIndex` $1 \dots 16$).
  5. **Paired Temperature Rule Verified**:
     - Cell 1 & 2 $\rightarrow$ `Temp#01`
     - Cell 3 & 4 $\rightarrow$ `Temp#02`
     - Cell 5 & 6 $\rightarrow$ `Temp#03`
     - Cell 7 & 8 $\rightarrow$ `Temp#04`
  6. Temperature values stored in both Celsius (`TemperatureC`) and Fahrenheit (`TemperatureF`).
  7. In `Battery.BatteryMeasure`:
     - 128 records created.
     - `val_bat_measure_point_float_voltage` populated (Online = `true`).
     - `val_bat_measure_point_offline_open_circuit_voltage` = `empty`.
     - `val_bat_measure_point_impedance` = `empty`.
     - `val_bat_measure_point_resistance` = `empty`.
* **Pass/Fail Criteria**: Exactly 128 measures committed; latest date stamp row selected; no NaN or null voltages.

#### TC-BAT-02: Samsung SDI BATTMON 136S Configuration (17 Modules x 8 Cells)
* **Scenario ID**: `TC-BAT-02`
* **Objective**: Verify 17-module system (136 cells) and 3-decimal clamping.
* **Preconditions**: `jarsPerBattery` = `17`, `cellsPerJar` = `8`, `measurementPoints` = `136`.
* **Input File**: `RACK001_136S_Log_CELL.csv`.
* **Execution Parameters**: `jarsPerBattery = 17`, `measurementPoints = 136`, `rack = "1"`.
* **Expected Result**: 17 modules created, each with 8 cells; total 136 cells; voltages clamped to 3 decimal places (e.g. `3.824 V`).

#### TC-BAT-03: Samsung SDI BATTMON Rack Mismatch Guard
* **Scenario ID**: `TC-BAT-03`
* **Objective**: Verify that attempting to upload a Rack 1 BATTMON file when Rack 2 was requested triggers the integrity guard.
* **Preconditions**: Asset form configured for Rack 2 (`st_bat_rack_no = "2"`).
* **Input File**: `RACK001_Log_CELL_0.csv` (Designated rack in header is `[RACK#001]`).
* **Execution Parameters**: `rack = "2"`.
* **Expected Result**:
  1. Action halts immediately.
  2. `BatteryData.ErrorString` populated with:  
     `"Rejected: BATTMON file is RACK#001 but rack 2 was requested"`.
  3. No `Module` or `CellInfo` records created.
  4. Nanoflow displays error alert to technician.

---

### Type 2: Samsung SDI ELPMON (`SAMSUNG_ELPMON`)

#### TC-ELP-01: Samsung SDI ELPMON Multi-Rack Column Isolation
* **Scenario ID**: `TC-ELP-01`
* **Objective**: Verify that when uploading a multi-rack ELPMON sheet (containing columns for R#1, R#2, and R#3), passing `rack: "2"` isolates only Rack 2 data.
* **Preconditions**: Multi-rack spreadsheet with columns `R#1/M#01_Cell01[V]`, `R#2/M#01_Cell01[V]`, `R#3/M#01_Cell01[V]`.
* **Execution Parameters**: `jarsPerBattery = 16`, `measurementPoints = 128`, `rack = "2"`.
* **Expected Result**:
  1. `BatteryData.BatteryType` = `Battery.Enum_BatteryType.SAMSUNG_ELPMON`.
  2. `BatteryData.Rack` = `"2"`.
  3. Only columns matching prefix `R#2/` are extracted.
  4. Columns for `R#1` and `R#3` are completely discarded.
  5. 16 modules $\times$ 8 cells = 128 cells extracted.

#### TC-ELP-02: Samsung SDI ELPMON Unused Channel Filtering (Cell09..Cell12 Truncation)
* **Scenario ID**: `TC-ELP-02`
* **Objective**: Verify that trailing pre-allocated columns `Cell09[V]` through `Cell12[V]` containing hyphens (`-`) or empty cells are discarded without creating phantom records.
* **Input File**: `20200702_132519_ELP_MON_SYSTEM_CELL_Log_#R1-#R10_0.csv`.
* **Execution Parameters**: `jarsPerBattery = 16`, `measurementPoints = 128`, `rack = "1"`.
* **Expected Result**:
  1. Trailing columns `Cell09` .. `Cell12` are skipped.
  2. Extracted cell count is exactly `128` (not 192).
  3. `BatteryData.TotalCellCount` = `128`.
  4. No phantom cells with null or zero voltages instantiated.

---

### Type 3: Samsung System Summary Log Guard (`SAMSUNG_SYSTEM_LOG`)

#### TC-SYS-01: Rejection of Samsung System Summary Log
* **Scenario ID**: `TC-SYS-01`
* **Objective**: Verify that if a technician accidentally uploads a Samsung System Summary file (`_SYSTEM_Log_0.csv`) containing `[SYSTEM]` but lacking cell voltages, the import halts with an actionable error.
* **Input File**: `20200702_132519_ELP_MON_SYSTEM_Log_#R1-#R10_0.csv`.
* **Execution Parameters**: `jarsPerBattery = 16`, `measurementPoints = 128`.
* **Expected Result**:
  1. Intercepted as `SAMSUNG_SYSTEM_LOG`.
  2. `BatteryData.ErrorString` populated with:  
     `"Validation Error: '<FileName>' is a Samsung BMS System Summary log and does not contain individual cell readings. Please upload the corresponding Cell Log file (e.g. '*_SYSTEM_CELL_Log_*.csv' or '*_Log_CELL_*.csv')."`
  3. No `Battery.BatteryMeasure` records created or deleted.
  4. Transaction rolled back via `SUB_Battery_CleanupFailedUpload`.

---

### Type 4: HPL / Vertiv VEC Lithium Snapshots (`HPL_VEC_SNAPSHOT`)

#### TC-HPL-01: Vertiv Li7 22S3P Layout (6 Modules x 22 Cells = 132 Cells)
* **Scenario ID**: `TC-HPL-01`
* **Objective**: Verify HPL 22S layout with MBB triplet sensor mapping (PCB Temp, Busbar Temp, Cell Temp).
* **Input File**: `PM bms_snapshot.txt` (Text file containing `Snapshot for RACK SERIAL`, `MBB Triplet` readings).
* **Execution Parameters**: `jarsPerBattery = 6`, `measurementPoints = 132` ($6 \times 22$), `rack = "1"`.
* **Expected Result**:
  1. `BatteryData.BatteryType` = `Battery.Enum_BatteryType.HPL_VEC_SNAPSHOT`.
  2. `BatteryData.RackSerial` parsed from file header.
  3. 6 `Module` entities instantiated (`ModelIndex = 1 .. 6`).
  4. Each module has 22 cells (`CellIndex = 1 .. 22`).
  5. Total cells = `132`.
  6. **MBB Triplet Sensor Assignment Verified**:
     - 2 MBBs per module $\times$ 11 cells each.
     - Sensor index 2 (Cell Temp) mapped to `CellInfo.TemperatureC` and `TemperatureF`.
     - Sensor index 0 mapped to `pcbTemp`.
     - Sensor index 1 mapped to `busBarTemp`.

#### TC-HPL-02: HPL 9S3P Layout (16 Modules x 9 Cells = 144 Cells)
* **Scenario ID**: `TC-HPL-02`
* **Objective**: Verify thermal cycling rule `cycle-skip-last` where sensor 4 is module temp and sensors 1-3 cycle across cells 1-9.
* **Execution Parameters**: `jarsPerBattery = 16`, `measurementPoints = 144` ($16 \times 9$).
* **Expected Result**: 16 modules $\times$ 9 cells; sensor 1 maps to cells 1, 4, 7; sensor 2 maps to cells 2, 5, 8; sensor 3 maps to cells 3, 6, 9; sensor 4 preserved as module temperature.

#### TC-HPL-03: HPL Module Health Accumulators Verification
* **Scenario ID**: `TC-HPL-03`
* **Objective**: Verify extraction of module operational tracking parameters.
* **Expected Result**: For each module, attributes populated:
  - `module_serno`
  - `mbb_serno`
  - `mbb_partno`
  - `dwell_20C`
  - `dwell_32C`
  - `temp_history`
  - `soc_history`
  - `kwh_thru_module`

---

### Type 5: Lithium Pack BMS Multi-Pack Logs (`LITHIUM_PACK_BMS`)

#### TC-LIT-01: Lithium Pack BMS Regrouping (7 Jars x 22 Cells vs. 22 Half-Packs)
* **Scenario ID**: `TC-LIT-01`
* **Objective**: Verify the Base Rule fix on multi-pack lithium CSV exports where raw file contains 22 half-packs of 7 cells, but user configuration specifies 7 jars with 22 cells/jar (154 points).
* **Preconditions**:
  - `batterytype` = `'Lithium'`
  - `dc_source_bat_string_jars_blocs_per_string` = `7`
  - `dc_source_bat_string_cells_per_bloc` = `'22'`
  - `measurementPoints` = `154`
* **Input File**: `120200_PACK(B12-R02)_20260525.csv`.
* **Execution Parameters**:
  - `jarsPerBattery` = `7`
  - `measurementPoints` = `154`
  - `rack` = `"1"`
  - `isModuleUpload` = `false`
  - `shouldValidateCellCount` = `false`
* **Expected Result**:
  1. Raw 154 cells are flattened across all half-packs.
  2. Regrouped into **exactly 7 Module entities** (`ModelIndex = 1 .. 7`).
  3. Each module contains **exactly 22 CellInfo entities** (`CellIndex = 1 .. 22`).
  4. Cell sequence verified:
     - Module 1: `ModelIndex = 1`, `CellIndex = 1 .. 22`, `cellNo = "1" .. "22"`.
     - Module 7: `ModelIndex = 7`, `CellIndex = 1 .. 22`, `cellNo = "133" .. "154"`.
  5. Module names formatted as `${rackSerial}_M${m}` (e.g. `120200_PACK(B12-R02)_M1`).
  6. **Nanoflow Loop Verification**:
     - `SUB_Battery_ParseAndValidateBMS` loops `ModuleIndex 1 .. 7` and `CellIndex 1 .. 22`.
     - **NO** `"MEASUREMENT POINT MISSING FOR MODULE 1 Cell 8"` error thrown!
  7. In `Battery.BatteryMeasure`:
     - Exactly 154 records committed.
     - `val_bat_block_data_table` = `1 .. 7`.
     - `val_bat_measure_point_no_data_table` = `1 .. 22`.

#### TC-LIT-02: Lithium Pack BMS Inverted Geometry (22 Jars x 7 Cells)
* **Scenario ID**: `TC-LIT-02`
* **Objective**: Verify bidirectional adaptability when technician configures 22 jars with 7 cells/jar.
* **Preconditions**: `jarsPerBattery` = `22`, `cellsPerJar` = `7`, `measurementPoints` = `154`.
* **Input File**: `120200_PACK(B12-R02)_20260525.csv`.
* **Execution Parameters**: `jarsPerBattery = 22`, `measurementPoints = 154`.
* **Expected Result**: Exactly 22 modules created with 7 cells each (`CellIndex = 1 .. 7`, `ModelIndex = 1 .. 22`); Nanoflow validation passes completely.

---

### Type 6: Alber BDSU Backup Archives & Databases (`ALBER_BDSU`)

#### TC-BDS-01: Full Alber BDSU `.bkup` Archive Extraction
* **Scenario ID**: `TC-BDS-01`
* **Objective**: Verify in-memory zip decompression of `.bkup` archive, SQLite B-tree leaf parsing of `bdsu_cfg` & `bdsu_dat`, and equipmentTag matching.
* **Input File**: `BDSU_Backup_StationA.bkup` (Contains `bdsu_cfg` and `bdsu_dat`).
* **Execution Parameters**: `jarsPerBattery = 60`, `measurementPoints = 60`, `equipmentTag = "UPS-STRING-1"`.
* **Expected Result**:
  1. `BatteryData.BatteryType` = `Battery.Enum_BatteryType.ALBER_BDSU`.
  2. Successfully parses without requiring native C/C++ SQLite binaries.
  3. Matches `equipmentTag` to `StringID: 1` in `BAMConfiguration`.
  4. Selects latest time-series record from `StringHistory` and `ResistanceHistoryFile`.
  5. 60 cells extracted (`CellIndex = 1 .. 60`).
  6. DC Ohmic resistance stored in `val_bat_measure_point_resistance` in raw $\mu\Omega$.
  7. `val_bat_measure_point_impedance` = `empty` (Ohmic separation rule).

#### TC-BDS-02: Alber BDSU Binary BLOB Decoding Modulo 256
* **Scenario ID**: `TC-BDS-02`
* **Objective**: Verify modulo 256 decoding and escape sequence replacement (`0101` $\to$ `00`, `0102` $\to$ `01`, `0103` $\to$ `27`).
* **Expected Result**:
  - Raw BLOB voltage `2250` divided by 1000 $\to$ `2.250 V`.
  - Raw BLOB temperature `770` divided by 10 $\to$ `77.0 °F`, converted to `25.0 °C`.
  - Resistance stored directly without 1000x multiplier.

#### TC-BDS-03: Alber BDSU Dual-Purpose Rack Fallback
* **Scenario ID**: `TC-BDS-03`
* **Objective**: Verify that when `equipmentTag` is omitted, passing `rack: "2"` selects `StringID: 2`.
* **Execution Parameters**: `equipmentTag = empty`, `rack = "2"`.
* **Expected Result**: Parser extracts data corresponding to `StringID: 2` from SQLite tables.

#### TC-BDS-04: Alber BDSU Configuration-Only DB Guard
* **Scenario ID**: `TC-BDS-04`
* **Objective**: Verify rejection when a user uploads `bdsu_cfg.db` directly.
* **Input File**: `bdsu_cfg.db`.
* **Expected Result**: Halts with: `"Validation Error: File 'bdsu_cfg.db' is a BDSU configuration database (bdsu_cfg) and does not contain measurement readings. Please upload the full .bkup backup archive ... or upload 'bdsu_dat.db'."`.

---

### Type 7: Alber Cellcorder / CDF (`ALBER_CDF`)

#### TC-CDF-01: Alber Cellcorder `.CDF` Standard Parsing
* **Scenario ID**: `TC-CDF-01`
* **Objective**: Verify parsing of ASCII `.CDF` inspection log, converting millivolts to volts and mapping resistance to DC Ohmic resistance.
* **Input File**: `SITE01_CRT400.cdf` (Contains `#SI`, `#MN CRT-400`, `#RD`, `#CD` rows).
* **Execution Parameters**: `jarsPerBattery = 60`, `measurementPoints = 60`.
* **Expected Result**:
  1. `BatteryData.BatteryType` = `Battery.Enum_BatteryType.ALBER_CDF`.
  2. Column 2 voltage (e.g. `13450 mV`) divided by 1000 $\to$ `13.450 V`.
  3. Column 3 internal resistance mapped directly to `val_bat_measure_point_resistance` in $\mu\Omega$.
  4. `val_bat_measure_point_impedance` = `empty`.
  5. 60 cells instantiated.

#### TC-CDF-02: Firmware 256-Cell Padding Truncation Rule
* **Scenario ID**: `TC-CDF-02`
* **Objective**: Verify that rows 61 to 256 containing firmware pre-allocation padding (`00000` resistance) are truncated.
* **Input File**: `.CDF` file with 60 valid cell readings followed by rows 61..256 with `#CD nn 00000 00000`.
* **Expected Result**: Parser halts reading at row 61 upon seeing `00000`; exactly 60 cells returned; total cell count = 60.

---

### Type 8: Hioki BT3554 / BT3554-50 Battery HiTester (`HIOKI_BT3554`)

#### TC-HIO-01: Hioki BT3554 Standard Metadata & Impedance Ingestion
* **Scenario ID**: `TC-HIO-01`
* **Objective**: Verify extraction of `#` metadata headers, memory bank tags, and AC impedance scaled to $m\Omega$ ($\times 1000$).
* **Input File**: `BT3554_String1.csv` (Headers `#Type = LEAD-ACID`, `#Model No. = BT3554-50`, `#Memory = UPS_1`, table starting with `No.`).
* **Execution Parameters**: `jarsPerBattery = 40`, `measurementPoints = 40`.
* **Expected Result**:
  1. `BatteryData.BatteryType` = `Battery.Enum_BatteryType.HIOKI_BT3554`.
  2. Model `BT3554-50` and Memory tag `UPS_1` extracted.
  3. AC impedance scaled by 1000 and stored in `val_bat_measure_point_impedance`.
  4. `val_bat_measure_point_resistance` = `empty`.
  5. 40 records created in `Battery.BatteryMeasure`.

---

### Type 9: PowerShield Continuous Monitor (`HIOKI_BT3554` Mapped)

#### TC-POW-01: PowerShield Standardization into Hioki Pipeline
* **Scenario ID**: `TC-POW-01`
* **Objective**: Verify that selecting `FileUploadType = 'PowerShield file'` maps to `Hioki file` and parses cleanly through the `HIOKI_BT3554` engine.
* **Preconditions**: `st_select_file_type_for_import` = `'PowerShield file'`.
* **Input File**: PowerShield export CSV.
* **Expected Result**:
  1. `SUB_Battery_ResolveUploadContext` maps `FileUploadType` to `'Hioki file'`.
  2. `ReadingInstrument` derived as `'Hioki'`.
  3. File parses successfully under `HIOKI_BT3554`.
  4. AC impedances and voltages mapped into `Battery.BatteryMeasure`.

---

### Type 10: Hioki 3554 Legacy HiTester (`HIOKI_3554_LEGACY`)

#### TC-HIO-02: Legacy Hioki 3554 CSV Disambiguation
* **Scenario ID**: `TC-HIO-02`
* **Objective**: Verify that legacy CSV exports without `#` headers filter out comparator judgment columns (`Voltage Judgement`, `Resistance Limit`).
* **Input File**: Legacy 3554 CSV containing `3554 BATTERY HiTESTER`.
* **Expected Result**: Extracted measurements contain only actual measured voltage, resistance, and temperature; comparator limits discarded.

---

### Type 11: Fluke BT500 / BT510 / BT521 Analyzer (`FLUKE`)

#### TC-FLU-01: Fluke Battery Analyzer Tabular Import
* **Scenario ID**: `TC-FLU-01`
* **Objective**: Verify Fluke analyzer CSV/XLSX export parsing cell discharge and internal resistance tables.
* **Input File**: `FLUKE IR METER RAW DATA.csv`.
* **Expected Result**: Cell voltages, temperatures, and internal impedances extracted and mapped to `Battery.BatteryMeasure`.

---

### Type 12: ZincFive Nickel-Zinc BMS Logs (`ZINC`)

#### TC-ZIN-01: ZincFive Time-Series Multi-Column Extraction (xlUp Bottom Scan)
* **Scenario ID**: `TC-ZIN-01`
* **Objective**: Verify ZincFive 100+ column spreadsheet import, scanning from bottom row (`xlUp`) for latest timestamp, extracting cabinet metrics and dynamic cell columns.
* **Input File**: `ZincFive_String_Log.csv` (Contains `Battery Voltage 1..40`, `System SOC`, `System SOH`).
* **Expected Result**:
  1. Latest populated chronological row selected.
  2. `SystemSOC`, `SystemSOH`, `SystemVoltage`, `SystemCurrent` populated on `BatteryData`.
  3. Cell columns `Battery Voltage 1..N` extracted in ascending order.
  4. Temperatures converted from Celsius to Fahrenheit.

---

### Type 13: Generic Tabular Fallback (`GENERIC_TABULAR`)

#### TC-TAB-01: Generic Tabular CSV Import
* **Scenario ID**: `TC-TAB-01`
* **Objective**: Verify fallback parser dynamically discovers column headers (`cell`, `voltage`, `temp`, `resistance`) in first 20 rows.
* **Input File**: `Generic_Battery_Readings.csv`.
* **Expected Result**: Columns mapped to `CellInfo` and synchronized into `Battery.BatteryMeasure`.

#### TC-TAB-02: Generic Tabular Incompatible File Rejection
* **Scenario ID**: `TC-TAB-02`
* **Objective**: Verify rejection when a user uploads an unrelated spreadsheet (e.g. employee list or invoice).
* **Input File**: `Unrelated_Document.csv` (No voltage or cell headers).
* **Expected Result**: Halts with: `"Validation Error: File '...' is not recognized as a valid BMS measurements file. Could not find header columns for cell voltage or readings."`.

---

## 3. Module Upload Mode (`isModuleUpload = true`) Test Cases

### TC-MOD-01: Valid Module Upload Mode (Hioki 40 Jars)
* **Scenario ID**: `TC-MOD-01`
* **Objective**: Verify that when `isModuleUpload = true`, each row is treated as a module reading ($N$ rows = $N$ modules) with `CellIndex = 1`.
* **Preconditions**:
  - `jarsPerBattery` = `40`
  - `cellsPerJar` = `6`
  - `measurementPoints` = `240` ($40 \times 6$)
  - `isModuleUpload` = `true`
* **Input File**: `Hioki_Module_Readings_40.csv` (Contains exactly 40 reading rows).
* **Execution Parameters**: `jarsPerBattery = 40`, `isModuleUpload = true`.
* **Expected Result**:
  1. `BatteryData.IsModuleUpload` = `true`.
  2. Total extracted cells = `40` (matches `jarsPerBattery`).
  3. Exactly 40 `Module` entities created (`ModelIndex = 1 .. 40`).
  4. Each module has exactly 1 `CellInfo` with `CellIndex = 1`.
  5. In `SUB_Battery_SyncBatteryMeasures`:
     - Reads mapped to `CellIndex = 1` for each module.
     - **Module-Only Padding Loop**: Generates placeholder empty records for cells 2 through 6 for each of the 40 modules (total 240 records in database).
     - Technicians see full 240-cell grid with module float voltages on cell 1 and clean blanks for cells 2..6.

### TC-MOD-02: Multi-Cell File Rejection in Module Upload Mode
* **Scenario ID**: `TC-MOD-02`
* **Objective**: Verify that attempting to upload a file with detailed multi-cell readings when `isModuleUpload = true` is immediately rejected.
* **Preconditions**: `isModuleUpload = true`, `jarsPerBattery = 16`.
* **Input File**: `RACK001_Log_CELL_0.csv` (Samsung BATTMON with 8 cell columns per module).
* **Execution Parameters**: `isModuleUpload = true`.
* **Expected Result**:
  1. Detected multi-cell structure (`hasMoreThanOneCellPerJar = true`).
  2. Immediately halts and throws:  
     `"Float voltages are not measured at each measure point. Kindly upload a file which only has one reading for each jar[Module]"`.
  3. No database changes committed.

### TC-MOD-03: Reading Count Mismatch in Module Upload Mode
* **Scenario ID**: `TC-MOD-03`
* **Objective**: Verify that if a module upload file has 38 rows but `jarsPerBattery = 40`, import fails.
* **Execution Parameters**: `jarsPerBattery = 40`, `isModuleUpload = true`, file has 38 rows.
* **Expected Result**: Fails with reading count mismatch ($38 \ne 40$).

---

## 4. Multiple File Upload & Re-Upload Scenarios (Lifecycle & Cross-Format Transitions)

This section details complex lifecycle transitions where technicians upload multiple files in succession to the same battery string questionnaire.

### Scenario A: Alber CDF Upload Followed by Module-Only Hioki Re-Upload
* **Scenario ID**: `TC-MULTI-01`
* **Objective**: Verify transition from detailed DC Ohmic resistance readings (Alber CDF) to module-level offline float voltages (Hioki).
* **Execution Sequence**:
  1. **Upload 1 (Alber CDF)**:
     - Technician uploads `SITE01_CRT400.cdf` (60 jars $\times$ 1 cell = 60 cells).
     - Domain state: 60 `BatteryMeasure` records created.
     - `val_bat_measure_point_float_voltage` = `13.450 V`.
     - `val_bat_measure_point_resistance` = `1250 µΩ`.
     - `val_bat_measure_point_impedance` = `empty`.
     - Access status = `LinkedOut: Y`.
  2. **Upload 2 (Hioki Module-Only Re-Upload)**:
     - Technician subsequently selects `st_select_file_type_for_import = 'Hioki file'`, checks module upload (`isModuleUpload = true`), and uploads `Hioki_60Jars.csv`.
* **Expected Result**:
  1. `SUB_Battery_SyncBatteryMeasures` retrieves existing 60 measures into `$AllBatteryMeasureListOld`.
  2. In-memory matching updates the 60 records:
     - `val_bat_measure_point_offline_open_circuit_voltage` = updated with Hioki voltage.
     - `val_bat_measure_point_float_voltage` = set to `empty` (Offline mode).
     - `val_bat_measure_point_impedance` = updated with Hioki AC impedance ($\times 1000$).
     - `val_bat_measure_point_resistance` = set to `empty` (Ohmic separation).
  3. Batch commit executes; set difference (`$MeasuresToDelete`) is empty (0 deleted).
  4. Resulting database state has exactly 60 records with updated Hioki attributes and clean nulls for obsolete Alber fields.

### Scenario B: Module-Only Hioki Upload Followed by Full Alber CDF Re-Upload
* **Scenario ID**: `TC-MULTI-02`
* **Objective**: Verify transition from padded placeholder records to full detailed multi-cell measurements.
* **Execution Sequence**:
  1. **Upload 1 (Hioki Module-Only with Padding)**:
     - 40 jars $\times$ 6 cells configured (240 points).
     - Technician uploads module file (40 readings).
     - System creates 40 measured records (cell 1) + 200 padded placeholder records (cells 2..6 with empty voltage/impedance).
  2. **Upload 2 (Full Alber CDF File)**:
     - Technician obtains full test set and uploads `Full_Alber_240Cells.cdf`.
* **Expected Result**:
  1. Full file contains 240 valid cell readings.
  2. In-memory update maps all 240 records directly.
  3. All 200 placeholder records are overwritten with real cell voltages and DC resistances.
  4. Total committed records = `240`; zero placeholder records remain.

### Scenario C: Lithium BMS Log Upload Followed by Portable Hioki Module-Level Upload
* **Scenario ID**: `TC-MULTI-03`
* **Objective**: Verify chemistry behavior when transitioning between online BMS logs and offline portable testers on a Lithium string.
* **Execution Sequence**:
  1. **Upload 1 (BMS Log)**: Uploads `RACK001_Log_CELL_0.csv` (16 jars $\times$ 8 cells = 128 cells).
     - `IsLithium = true`, `IsOnline = true`.
     - Measures populated with Online Float Voltage (`val_bat_measure_point_float_voltage`).
  2. **Upload 2 (Hioki Tester)**: Uploads `Hioki_16Modules.csv` (16 readings).
     - `Sub_CalculateifOnlyModuledataImport` evaluates `true` for Hioki on Lithium.
     - `Sub_CheckOnlineOfOfflineVoltage` evaluates `false` (Offline voltage).
* **Expected Result**:
  1. Readings mapped to module terminal cells (`CellIndex = 1`).
  2. Stored in `val_bat_measure_point_offline_open_circuit_voltage`.
  3. `val_bat_measure_point_float_voltage` reset to `empty`.
  4. Cells 2..8 padded with clean placeholders.

### Scenario D: Multi-Rack Sequential Uploads (Rack 1 then Rack 2)
* **Scenario ID**: `TC-MULTI-04`
* **Objective**: Verify complete data isolation across multiple battery strings/racks in the same job.
* **Execution Sequence**:
  1. Technician opens String 1 (Rack 1) questionnaire wizard and uploads Rack 1 file (`RACK001_Log_CELL.csv`).
  2. Verifies 128 measures created under `QuestionnaireId_1`.
  3. Technician navigates to String 2 (Rack 2) questionnaire wizard and uploads Rack 2 file (`RACK002_Log_CELL.csv`).
* **Expected Result**:
  1. Rack 2 measures are associated with `QuestionnaireId_2`.
  2. Database query for String 1 returns exactly 128 measures (Rack 1).
  3. Database query for String 2 returns exactly 128 measures (Rack 2).
  4. Zero cross-talk or overwriting across questionnaires.

### Scenario E: Consecutive Upload of Exact Same File (Idempotency)
* **Scenario ID**: `TC-MULTI-05`
* **Objective**: Verify that re-uploading the exact same file twice produces an identical database state with zero duplicates.
* **Execution Sequence**: Upload `Hioki_String1.csv` $\rightarrow$ inspect $\rightarrow$ re-upload `Hioki_String1.csv`.
* **Expected Result**:
  1. In Phase 7, `$AllBatteryMeasureListOld` matches `$BatteryMeasureList`.
  2. Each record is updated in-place via `SUB_BatteryMeasure_CreateOrUpdate`.
  3. Set subtraction `$MeasuresToDelete` is empty.
  4. Final record count in database remains exactly `40`; zero duplicates created.

### Scenario F: Format Cross-Validation Mismatch Guard
* **Scenario ID**: `TC-MULTI-06`
* **Objective**: Verify that selecting "Hioki file" in questionnaire dropdown but uploading a Samsung BATTMON CSV triggers the cross-validation error guard.
* **Preconditions**: Dropdown `st_select_file_type_for_import` = `'Hioki file'`.
* **Input File**: `RACK001_Log_CELL_0.csv` (Detected as `SAMSUNG_BATTMON`).
* **Expected Result**:
  1. JavaScript Action parses file successfully and returns `BatteryType = SAMSUNG_BATTMON`.
  2. Nanoflow `SUB_Battery_ParseAndValidateBMS` evaluates `same FileTypes?` decision.
  3. Identifies mismatch: selected `'Hioki file'` $\ne$ uploaded `'SAMSUNG_BATTMON'`.
  4. Displays modal warning:  
     `"Type of file Uploaded SAMSUNG_BATTMON and selected Hioki file do not match. Upload failed."`
  5. Calls `SUB_Battery_CleanupFailedUpload`:
     - Deletes uploaded file attachment.
     - Existing database records remain untouched.
  6. Ingestion halts safely.

### Scenario G: Battery Geometry Reconfiguration Mid-Job
* **Scenario ID**: `TC-MULTI-07`
* **Objective**: Verify that if a technician reconfigures the battery string geometry (e.g. from 40 jars $\times$ 6 cells to 30 jars $\times$ 8 cells) and re-uploads, obsolete records are purged.
* **Execution Sequence**:
  1. Initial config: `jars = 40`, `cells = 6` (240 points) $\rightarrow$ 240 records created (`ModelIndex 1..40`, `CellIndex 1..6`).
  2. Reconfiguration: Technician changes questionnaire to `jars = 30`, `cells = 8` (240 points) and re-uploads.
* **Expected Result**:
  1. New upload generates 240 records with `ModelIndex 1..30` and `CellIndex 1..8`.
  2. Phase 7 matches available combinations in-memory.
  3. Records with `ModelIndex > 30` or `CellIndex > 8` are identified in set difference `$MeasuresToDelete`.
  4. Obsolete records are batch-deleted (`delete $MeasuresToDelete refresh`).
  5. Final database state contains exactly 240 records matching the 30 $\times$ 8 geometry.

### Scenario H: Corrupted / Partial File Upload Rollback
* **Scenario ID**: `TC-MULTI-08`
* **Objective**: Verify transactional rollback when an upload fails partway through validation.
* **Execution Sequence**:
  1. String has 40 valid measures from a prior upload.
  2. Technician attempts to re-upload a corrupted 0-byte file or a file with mismatched cell count.
* **Expected Result**:
  1. Validation fails in Phase 6 (`SUB_Battery_ParseAndValidateBMS`).
  2. Action triggers `SUB_Battery_CleanupFailedUpload`.
  3. Failed upload media deleted.
  4. Previously imported 40 valid records in database are **retained** without corruption.

---

## 5. Edge Cases, Boundary Conditions & System Faults

| Test Case ID | Test Scenario | Input / Precondition | Expected Behavior / Output |
| :--- | :--- | :--- | :--- |
| **TC-EDGE-01** | **0-Byte Empty File** | File size = 0 bytes | Halts immediately with: `"Validation Error: The selected file '...' is empty (0 bytes)."`. Zero entities created. |
| **TC-EDGE-02** | **Non-Divisible Geometry** | `jars = 7`, `points = 155` ($155 / 7 = 22.14$) | Immediate exit with divisibility error: `"Validation Error: Measurement points (155) must be divisible by jars per battery (7)..."`. |
| **TC-EDGE-03** | **Decoupled Validation Mode** | `shouldValidateCellCount = false`, 120 cells vs 128 expected | Action does NOT throw red-box error; populates `BatteryData.IsCellCountValid = false`; returns parsed tree for Nanoflow inspection. |
| **TC-EDGE-04** | **Strict Validation Override** | `shouldValidateCellCount = true`, 120 cells vs 128 expected | Action throws runtime error or populates `BatteryData.ErrorString`. |
| **TC-EDGE-05** | **Null / Blank Cell Readings** | Cell voltage cell is blank or `#N/A` | Reading stored as `empty` rather than `0` (prevents skewing string averages and baselines); `val_bat_measure_point_access` set to `LinkedOut = Y`. |
| **TC-EDGE-06** | **Negative Voltage Filtering** | Cell voltage is `-0.05 V` | Value clamped to `empty` or flagged with custom error; logged in diagnostic output. |
| **TC-EDGE-07** | **Extreme Thermal Readings** | Temperature $> 100^\circ\text{C}$ or $< -40^\circ\text{C}$ | Flagged as open/shorted probe; stored as null with diagnostic warning. |
| **TC-EDGE-08** | **High-Volume Stress Test** | 50,000-row multi-rack ELPMON sheet (15 MB) | Parsed in browser memory in $< 500\text{ms}$; single bulk DB retrieve ($< 20\text{ms}$); batch DB commit ($< 50\text{ms}$); zero UI thread lockup. |
| **TC-EDGE-09** | **Special Characters in Filename** | `Log_CELL [B#12] (2026-05-25) & #1.csv` | File picker sanitizes name; regex patterns handle escaped brackets safely. |

---

## 6. UI Verification & `SNIP_ViewOrEdit_Measures` Test Cases

### DataGrid Rendering & Bulk Ingestion (`DS_GetBatteryMeasuresAdvanced`)
* **TC-UI-01**: Open dynamic questionnaire wizard to battery measures step.
* **Verification**:
  - `DS_GetBatteryMeasuresAdvanced` executes exactly 1 bulk SQLite retrieve.
  - DataGrid displays all string cells ordered by Block Number and Cell Number.
  - No noticeable lag or spinner freeze on mobile devices.

### DOM Injected Grouped Headers (`Incentro JavaScriptSnippet`)
* **TC-UI-02**: Inspect DataGrid DOM table header.
* **Verification**:
  - `.dg-group-cell` elements injected above standard headers.
  - **Group 1**: `IDENTIFICATION` covers columns 0 to 5.
  - **Group 2**: `MEASUREMENTS` covers columns 6 to 17.
  - **Group 3**: `TRACKING & THRESHOLDS` covers columns 18 to 21.
  - Group spans adjust dynamically if columns are hidden via questionnaire visibility rules.

### Sticky Frozen Columns Behavior & CSS Engine
* **TC-UI-03**: Scroll DataGrid horizontally.
* **Verification**:
  - Columns 0 through 5 remain pinned to the left edge of the viewport (`position: sticky; left: ...px`).
  - Column 5 (last frozen column) displays drop shadow (`box-shadow: 2px 0 4px rgba(0, 0, 0, 0.12)`).
  - Resizing the browser window triggers `ResizeObserver` to recalculate pixel offsets without overlapping.
  - Paging through the grid triggers `MutationObserver` to maintain sticky headers.

### In-Line Cell Editing & Dynamic Variance Commit Flow
* **TC-UI-04**: Edit a cell float voltage in row 5.
* **Verification**:
  - Clicking cell triggers `ACT_SetActiveMeasures_Row`.
  - Modifying voltage from `13.50` to `12.10` triggers `OCH_CommitBatteryMeasures`.
  - % Variance re-calculates dynamically against baseline.
  - Row badge shifts from Normal (Green) to Critical Low (Red).
  - Change is committed to client storage.

### Client-Side Chart Generation (`getMeasuresChartJSON.js`)
* **TC-UI-05**: View battery measures chart on client.
* **Verification**:
  - `getMeasuresChartJSON.js` executes without server call.
  - Renders Plotly bar chart with baseline multiplier bands (+30% Warning, +50% Critical).
  - Figma-styled hover card tooltip (`initCustomTooltip()`) displays Cell #, Value, Baseline, and Variance.
  - If string $> 50$ cells, dual-thumb range slider (`initRangeSelector()`) is mounted below chart.
  - Global cache `globalThis.__lastBatteryMeasures` populated for subsequent trending charts.

### Server-Side Java Chart Generation (`JA_GetMeasuresChartJSON`)
* **TC-UI-06**: Trigger PDF inspection report export.
* **Verification**:
  - Java action `JA_GetMeasuresChartJSON` calls `MeasuresChartGenerator.java`.
  - `RecordExtractor.java` extracts `BatteryMeasure` records, filtering out `LinkedOut` cells.
  - Generates valid `ChartDataJSON` and `ChartLayoutJson` strings on `Battery.ChartNPE`.
  - PDF document renders identical chart visualization matching the web client.

---

## 7. Traceability Matrix & Execution Checklist

| Test ID | Format / Component | Key Verification Gate | Automatable (Harness / Nanoflow) | Status |
| :--- | :--- | :--- | :---: | :---: |
| `TC-BAT-01` | Samsung BATTMON | Paired temps ($1/2\text{ cells}$), 128S | Automated (Node / Harness) | **PASS** |
| `TC-BAT-02` | Samsung BATTMON | 136S (17x8), 3-decimal clamping | Automated (Node / Harness) | **PASS** |
| `TC-BAT-03` | Samsung BATTMON | Rack mismatch integrity guard | Automated (Node / Harness) | **PASS** |
| `TC-ELP-01` | Samsung ELPMON | Multi-rack column isolation (`wantRack`) | Automated (Node / Harness) | **PASS** |
| `TC-ELP-02` | Samsung ELPMON | Trailing channel filtering (`Cell09..12`) | Automated (Node / Harness) | **PASS** |
| `TC-SYS-01` | Samsung System Log | System summary rejection guard | Automated (Node / Harness) | **PASS** |
| `TC-HPL-01` | HPL / Vertiv VEC | 22S3P (6x22), MBB triplet temperatures | Automated (Node / Harness) | **PASS** |
| `TC-HPL-02` | HPL / Vertiv VEC | 9S3P (16x9), `cycle-skip-last` temps | Automated (Node / Harness) | **PASS** |
| `TC-HPL-03` | HPL / Vertiv VEC | Module health accumulators extraction | Automated (Node / Harness) | **PASS** |
| `TC-LIT-01` | Lithium Pack BMS | 7x22 regrouping from 22 half-packs | Automated (Node / Harness) | **PASS** |
| `TC-LIT-02` | Lithium Pack BMS | 22x7 inverted geometry regrouping | Automated (Node / Harness) | **PASS** |
| `TC-BDS-01` | Alber BDSU Backup | Zip decompression, SQLite B-tree | Automated (Node / Harness) | **PASS** |
| `TC-BDS-02` | Alber BDSU Backup | Binary BLOB decoding modulo 256 | Automated (Node / Harness) | **PASS** |
| `TC-BDS-03` | Alber BDSU Backup | Dual-purpose rack fallback (`StringID`) | Automated (Node / Harness) | **PASS** |
| `TC-BDS-04` | Alber BDSU Backup | `bdsu_cfg.db` configuration guard | Automated (Node / Harness) | **PASS** |
| `TC-CDF-01` | Alber Cellcorder | ASCII `.CDF` import, DC resistance $\mu\Omega$ | Automated (Node / Harness) | **PASS** |
| `TC-CDF-02` | Alber Cellcorder | Firmware 256-cell padding truncation | Automated (Node / Harness) | **PASS** |
| `TC-HIO-01` | Hioki BT3554 | `#` metadata headers, AC impedance $m\Omega$ | Automated (Node / Harness) | **PASS** |
| `TC-POW-01` | PowerShield Monitor | Mapping to `HIOKI_BT3554` pipeline | Automated (Node / Harness) | **PASS** |
| `TC-HIO-02` | Hioki 3554 Legacy | Filtering comparator limits | Automated (Node / Harness) | **PASS** |
| `TC-FLU-01` | Fluke BT500 | Discharge & resistance table mapping | Automated (Node / Harness) | **PASS** |
| `TC-ZIN-01` | ZincFive NiZn | `xlUp` bottom-up scan, SOC/SOH | Automated (Node / Harness) | **PASS** |
| `TC-TAB-01` | Generic Tabular | Header auto-discovery in first 20 rows | Automated (Node / Harness) | **PASS** |
| `TC-TAB-02` | Generic Tabular | Rejection of non-measurement sheets | Automated (Node / Harness) | **PASS** |
| `TC-MOD-01` | Module Upload Mode | 40 module readings + 200 cell padding | Automated (Nanoflow) | **PASS** |
| `TC-MOD-02` | Module Upload Mode | Multi-cell file rejection guard | Automated (Node / Harness) | **PASS** |
| `TC-MOD-03` | Module Upload Mode | Reading count mismatch guard | Automated (Node / Harness) | **PASS** |
| `TC-MULTI-01`| Multi-File Lifecycle| Alber CDF $\to$ Hioki Module-Only | Manual / E2E Playwright | **PASS** |
| `TC-MULTI-02`| Multi-File Lifecycle| Hioki Module-Only $\to$ Full Alber CDF | Manual / E2E Playwright | **PASS** |
| `TC-MULTI-03`| Multi-File Lifecycle| Lithium BMS $\to$ Portable Hioki Tester | Manual / E2E Playwright | **PASS** |
| `TC-MULTI-04`| Multi-File Lifecycle| Multi-Rack Sequential Isolation | Manual / E2E Playwright | **PASS** |
| `TC-MULTI-05`| Multi-File Lifecycle| Idempotency (consecutive duplicate upload)| Manual / E2E Playwright | **PASS** |
| `TC-MULTI-06`| Multi-File Lifecycle| Format cross-check mismatch rollback | Manual / Nanoflow | **PASS** |
| `TC-MULTI-07`| Multi-File Lifecycle| Geometry change ($40\times 6 \to 30\times 8$) | Manual / Nanoflow | **PASS** |
| `TC-MULTI-08`| Multi-File Lifecycle| Corrupted file upload & safe rollback | Manual / Nanoflow | **PASS** |
| `TC-EDGE-01` | Edge Case | 0-Byte empty file | Automated (Node / Harness) | **PASS** |
| `TC-EDGE-02` | Edge Case | Non-divisible geometry ($155 / 7$) | Automated (Node / Harness) | **PASS** |
| `TC-EDGE-03` | Edge Case | `shouldValidateCellCount = false` | Automated (Node / Harness) | **PASS** |
| `TC-EDGE-05` | Edge Case | Null / Blank cell readings (LinkedOut) | Automated (Node / Harness) | **PASS** |
| `TC-UI-01`   | UI Presentation | Single bulk retrieve (`DS_Get...`) | Manual / Browser Profiler | **PASS** |
| `TC-UI-02`   | UI Presentation | Injected grouped headers (IDENT/MEAS) | Manual / Visual Inspector | **PASS** |
| `TC-UI-03`   | UI Presentation | Sticky frozen columns & CSS drop shadow | Manual / Visual Inspector | **PASS** |
| `TC-UI-04`   | UI Presentation | In-line cell editing & dynamic commit | Manual / Nanoflow | **PASS** |
| `TC-UI-05`   | UI Presentation | Client Plotly chart & custom tooltip | Manual / Browser Visual | **PASS** |
| `TC-UI-06`   | Java Chart Bridge | Server-side `JA_GetMeasuresChartJSON` | Automated (JUnit / Java) | **PASS** |
