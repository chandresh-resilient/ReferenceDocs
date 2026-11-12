# Appendix 04: Developer Reference & Code Atlas

> [!NOTE]
> **Parent Document**: [Connected CE V2 — Comprehensive Mendix Developer Knowledge Transfer (KT) Guide](file:///C:/Users/chand/.gemini/antigravity-cli/brain/675d50f8-c688-4e59-bfba-d5550106bbba/Connected_CE_V2_Comprehensive_KT_Guide.md)
>
> This appendix serves as the definitive reference manual and code atlas for Connected CE V2 developers. It indexes all mission-critical Microflows (MF), Nanoflows (NF), JavaScript Actions (JSA), Java Actions (JA), Pages, Snippets, and Entities, concluding with actionable step-by-step developer troubleshooting playbooks.

---

## 1. Master Microflow (MF) Reference Catalog

| Microflow Name | Module | Trigger / Invoker | Primary Responsibility & Parameters |
| :--- | :--- | :--- | :--- |
| **`SUB_SyncJobToDevice`** | `Core` | Server sync endpoint (`ACT_AppSync_Initiate`) | Serializes the complete job graph (`Job`, `Questionnaire`, `Pages`, `Questions`, `Options`, `Rules`, `HistoricalBaselines`) into a compressed JSON stream (`JobSyncFile`). |
| **`SUB_CreateQuestionnaire_Server`** | `Core` | Job Ingestion (`ACT_IngestJobPackage`) | Clones active configuration templates (`QuestionnaireConfig`, `PageConfig`, `QuestionConfig`) into operational runtime entities (`Questionnaire`, `Page`, `Question`). |
| **`ACT_Job_SyncToServer`** | `Core` | Engineer taps Sync Button or Network Reconnect | Validates network presence via `IsOnline.js`, extracts local changes from SQLite, constructs `JobToPush`, and submits to server REST endpoint. |
| **`ACT_ImportTagActionConfig`** | `DynamicQuestionnaire` | Admin UI: `TagActionConfig_Overview` | Reads uploaded Excel file, maps columns to `TagActionConfigImport`, uses `JA_QuestionnaireFormStringToEnum` for technology conversion, and commits `TagActionConfig` records. |
| **`ACT_ImportCalculateRules`** | `DynamicQuestionnaire` | Admin UI: `CalculateRule_Overview` | Ingests mathematical calculation rules from spreadsheet, checks formula AST syntax, and commits `CalculateRule` entities. |
| **`ACT_ImportOverrideRules`** | `DynamicQuestionnaire` | Admin UI: `OverrideRule_Overview` | Ingests conditional override rules, validates target `ImpactedTagId`, and commits `OverrideRule` entities. |
| **`ACT_ImportValidationRules`** | `DynamicQuestionnaire` | Admin UI: `ValidationRule_Overview` | Ingests OnPage and Popup validation constraints and links impacting questions. |
| **`ACT_IngestJobPackage`** | `RelayConnector` | HTTP POST from Vertiv Relay Bus | Entry point for enterprise work orders. Parses `SR_INFO_DETAIL_ITEM`, `UNIT_DETAILS_ITEM`, and `TASK_DETAILS_ITEM` into `Core.Job`. |
| **`SUB_CreateOrUpdateJob`** | `RelayConnector` | Upstream Sync (`ACT_Job_SyncToServer`) | Translates completed job data and measurements into the JSON payload expected by Oracle EBS and closes the work order. |
| **`SUB_GenerateJobReport_Server`** | `PDFs` | Customer Sign-off (`Act_EnQueForPrintAndCloseJob`) | Renders the customer-facing field service report PDF with embedded Plotly vector graphs and customer signature. |

---

## 2. Master Nanoflow (NF) Reference Catalog

| Nanoflow Name | Module | Trigger / Invoker | Primary Responsibility & Parameters |
| :--- | :--- | :--- | :--- |
| **`OC_Answer`** | `DynamicQuestionnaire` | On-Change event of every question input widget | The central answering pipeline. Writes typed value, updates timestamps, recalculates progress counters, evaluates visibility, and dispatches change impact. |
| **`SUB_ProcessChangeImpact`** | `DynamicQuestionnaire` | Called by `OC_Answer` when value changes | Orchestrates rule execution in sequence: Override Rules $\rightarrow$ Calculate Rules $\rightarrow$ Tag Actions $\rightarrow$ Validation Rules. |
| **`SUB_QuestionBehavior_Dispatcher`** | `DynamicQuestionnaire` | Invoked by `SUB_ProcessChangeImpact` and `JSA_CustomTagBehaviour_RunByImpacting` | Event bus for Tag Actions. Retrieves matching `TagActionConfig` rows by `ImpactingTagId` and Technology, looping over actions (`GenerateBatteryMeasures`, `SyncRackOptions`, etc.). |
| **`SUB_ApplyMassFormulaRules`** | `DynamicQuestionnaire` | Job Load / Page Render | Runs all rules across the entire questionnaire: `JSA_OverrideRules_RunByImpacted`, `JSA_CalculateRules_RunByImpacted`, `JSA_CustomTagBehaviour_RunByImpacting`, `JSA_ValidationRules_RunByImpacted`. |
| **`ValidationRule_CheckOneQuestion`** | `DynamicQuestionnaire` | Invoked by `JSA_ValidationRules_RunByImpacting` | Evaluates validation formula for a single question. Toggles `ValidationStatus` and applies error message if invalid. |
| **`SUBOCH_Battery_MeasuresData_Generate`** | `Battery` | Tag Action Dispatcher | Calculates total measurement points ($S \times B \times P$), expands or prunes `BatteryMeasure` entities, and preserves existing recorded values. |
| **`ACT_SyncRackOptions`** | `Battery` | Tag Action Dispatcher | Counts active battery SubQuestionnaires under the job and updates options on `st_bat_rack_no` dropdowns across all screens. |
| **`ACT_ResetStringBaseline`** | `Battery` | Tag Action Dispatcher (when `st_bat_replace_string` is checked) | Purges prior historical baselines for the replaced string and flags current visit readings as the new baseline benchmark. |
| **`ACT_Read_Hioki`** | `Battery` | Hioki Optical/Bluetooth Probe Snippet | Opens serial/BLE communication stream, captures voltage/resistance pairs from the handheld tester, and populates `BatteryMeasure` rows sequentially. |
| **`ACT_Upload_BatteryMeasures`** | `Battery` | NextPredict File Upload Snippet | Ingests NextPredict Excel or Cellcorder files, validates measurement envelopes, and updates the active string matrix. |
| **`ACT_Job_Open`** | `Core` | Engineer taps Job Card | Validates offline cache, sets user session context, and opens the root questionnaire. |
| **`ACT_Job_EditStatus_toCustomerReady`** | `Core` | Engineer marks job ready for sign-off | Calls `VAL_CustomerReady` to verify 100% completion of mandatory tags before locking editing and opening customer summary. |
| **`VAL_CustomerReady`** | `Core` | Invoked by `ACT_Job_EditStatus_toCustomerReady` | Traverses root questionnaire, battery strings, and thermal subquestionnaires to guarantee zero incomplete mandatory tags. |

---

## 3. Master JavaScript Action (JSA) Reference Catalog

| File Path / JSA Name | Module | Input Parameters | Return | Implementation Details |
| :--- | :--- | :--- | :---: | :--- |
| [`JSA_CustomTagBehaviour_RunByImpacting.js`](file:///C:/Users/chand/Mendix/CEV2-Devs/javascriptsource/dynamicquestionnaire/actions/JSA_CustomTagBehaviour_RunByImpacting.js) | `DynamicQuestionnaire` | `checkQuestionNanoflow`, `job`, `questionnaireIdCSV`, `changedTagId`, `pageListName` | `Boolean` | Executes offline SQL against `TagActionConfig`, joins `Question` and `Page`, matches technology, and invokes `SUB_QuestionBehavior_Dispatcher` for each matching question. |
| [`JSA_CalculateRules_RunByImpacting.js`](file:///C:/Users/chand/Mendix/CEV2-Devs/javascriptsource/dynamicquestionnaire/actions/JSA_CalculateRules_RunByImpacting.js) | `DynamicQuestionnaire` | `checkQuestionNanoflow`, `currentPage`, `job`, `questionnaireIdCSV`, `changedQuestion`, `pageListName` | `Boolean` | Finds Calculate Rules impacted by a changed tag via SQL and calls `executeCalculateRuleResult.js`. |
| [`JSA_CalculateRules_RunByImpacted.js`](file:///C:/Users/chand/Mendix/CEV2-Devs/javascriptsource/dynamicquestionnaire/actions/JSA_CalculateRules_RunByImpacted.js) | `DynamicQuestionnaire` | `checkQuestionNanoflow`, `job`, `questionnaireIdCSV`, `changedTagId`, `pageListName` | `Boolean` | Evaluates all Calculate Rules across a questionnaire during mass load (`SUB_ApplyMassFormulaRules`). |
| [`JSA_OverrideRules_RunByImpacting.js`](file:///C:/Users/chand/Mendix/CEV2-Devs/javascriptsource/dynamicquestionnaire/actions/JSA_OverrideRules_RunByImpacting.js) | `DynamicQuestionnaire` | `checkQuestionNanoflow`, `currentPage`, `job`, `questionnaireIdCSV`, `changedTagId`, `pageListName` | `Boolean` | Finds and executes Override Rules impacted by changed tags using direct SQLite queries. |
| [`JSA_ValidationRules_RunByImpacting.js`](file:///C:/Users/chand/Mendix/CEV2-Devs/javascriptsource/dynamicquestionnaire/actions/JSA_ValidationRules_RunByImpacting.js) | `DynamicQuestionnaire` | `checkQuestionNanoflow`, `currentPage`, `job`, `questionnaireIdCSV`, `changedQuestion`, `pageListName` | `Boolean` | Evaluates OnPage and Popup validation constraints in response to user inputs. |
| [`offlineSQLEngine.js`](file:///C:/Users/chand/Mendix/CEV2-Devs/javascriptsource/core/helpers/offlineSQLEngine.js) | `Core` | `sql` (String) | `Array<Object>` | Directly accesses Mendix offline database (`mx.data.offlineDataBackend._store._database.selectObjects`) for high-speed local SQL queries. |
| [`retrieveByGuid.js`](file:///C:/Users/chand/Mendix/CEV2-Devs/javascriptsource/core/helpers/retrieveByGuid.js) | `Core` | `guid` (String) | `MxObject` | Retrieves a Mendix object from local client cache by GUID via `mx.data.get`. |
| [`commitMxObjects.js`](file:///C:/Users/chand/Mendix/CEV2-Devs/javascriptsource/core/helpers/commitMxObjects.js) | `Core` | `mxObjects` (Array or Object) | `Promise<void>` | Commits an array of modified Mendix objects to the offline local store without triggering full-screen refresh loops. |
| [`getMeasuresChartJSON.js`](file:///C:/Users/chand/Mendix/CEV2-Devs/javascriptsource/battery/actions/getMeasuresChartJSON.js) | `Battery` | Measurement rows, threshold limits | `String (JSON)` | Constructs Plotly.js chart configuration (traces, bars, tolerance threshold lines) for the battery grid view. |
| [`JSA_GetBatteryTrendingChartData.js`](file:///C:/Users/chand/Mendix/CEV2-Devs/javascriptsource/battery/actions/JSA_GetBatteryTrendingChartData.js) | `Battery` | Current visit measures, historical baseline records | `String (JSON)` | Builds multi-year degradation curve comparing current cell resistance against commissioning baselines. |
| [`JSA_CheckIfJobReassigned.js`](file:///C:/Users/chand/Mendix/CEV2-Devs/javascriptsource/core/actions/JSA_CheckIfJobReassigned.js) | `Core` | `job` | `Boolean` | Verifies whether a work order was reassigned to another engineer in Oracle EBS while the local device was offline. |

---

## 4. Master Java Action (JA) Reference Catalog

| Java Action Class Name | Module | Input Parameters | Return | Purpose & Mechanics |
| :--- | :--- | :--- | :---: | :--- |
| **`JA_QuestionnaireFormStringToEnum`** | `DynamicQuestionnaire` | `QuestionnaireFormString` (String) | `Enum QuestionnaireForms` | Safely parses imported technology strings (e.g. `'UPS'`, `'Battery'`, `'Thermal'`) into enum values; evaluates invalid or empty strings to `null` (matching all technologies). |
| **`JA_CalculateResultTypeStringToEnum`**| `DynamicQuestionnaire` | `ResultTypeString` (String) | `Enum CalculateResultTypes` | Converts spreadsheet calculation types (`ValueString`, `ValueInteger`, `ValueDecimal`) to enum. |
| **`JA_GetStringMemberValue`** | `Core` | `TargetObject` (MxObject), `MemberName` (String) | `String` | Reflection helper extracting attribute values dynamically by name. |
| **`JA_GetMeasuresChartJSON`** | `Battery` | Battery data collection, chart config | `String (JSON)` | Server-side chart data generator used when compiling customer PDF reports. |
| **`CreateUUID`** | `DynamicQuestionnaire` | None | `String` | Generates RFC 4122 compliant UUIDs for client-generated entities. |
| **`endAllTransactions`** | `Battery` | None | `Boolean` | Emergency transaction cleanup rolling back dangling SQL locks during heavy batch imports. |

---

## 5. Master Page & Snippet (SNIP) Reference Catalog

| Page / Snippet Name | Module | UI Type | User Access Roles | Purpose & Content |
| :--- | :--- | :---: | :--- | :--- |
| **`TagActionConfig_Overview`** | `DynamicQuestionnaire` | Page | `ITAdmin`, `Configurator` | DataGrid of all configured Tag Actions with technology, action type, execution order, and Excel Import/Export buttons. |
| **`TagActionConfig_NewEdit`** | `DynamicQuestionnaire` | Page | `ITAdmin`, `Configurator` | Form for creating or modifying a Tag Action record, including optional `Technology` and `AdditionalParam`. |
| **`SNIP_Measured_IndividualPointValues`** | `Battery` | Snippet | `CustomerEngineer`, `ServicePartnerCE` | Primary battery inspection workspace: rack dropdown, float voltage sum, table/graph switcher, and cell data grid. |
| **`SNIP_BatteryRACK`** | `Battery` | Snippet | `CustomerEngineer`, `ServicePartnerCE` | Selector widget bound to `st_bat_rack_no`, allowing the technician to switch between active battery strings. |
| **`SNIP_MeasuresChart`** | `Battery` | Snippet | `CustomerEngineer`, `ServicePartnerCE` | Plotly.js chart displaying bar graphs of individual cell voltages and internal resistances. |
| **`SNIP_ViewOrEdit_Measures`** | `Battery` | Snippet | `CustomerEngineer`, `ServicePartnerCE` | Editable tabular matrix displaying Jar #, Cell #, Float V, Resistance, Strap R, and Temperature. |
| **`SNIP_Read_Hioki_Cellcorder`** | `Battery` | Snippet | `CustomerEngineer`, `ServicePartnerCE` | Interface for connecting handheld Hioki 3554 optical/Bluetooth testers and streaming readings. |
| **`SNIP_UploadBatteryMeasures`** | `Battery` | Snippet | `CustomerEngineer`, `ServicePartnerCE` | File upload modal supporting NextPredict Excel spreadsheets and Cellcorder CSV logs. |
| **`SNIP_BatteryDischarge`** | `Battery` | Snippet | `CustomerEngineer`, `ServicePartnerCE` | Capacity discharge test dashboard tracking time-series voltage drop under active load. |
| **`SNIP_SubQuestionnaire_Battery_New`**| `UPS` | Snippet | `CustomerEngineer`, `ServicePartnerCE` | SubQuestionnaire container isolating hardware parameters for an individual battery string. |
| **`Account_Overview_Custom`** | `Core` | Page | `ITAdmin`, `OperationalAdmin` | User management dashboard managing technician profiles, regions, and permissions. |
| **`DatabaseMonitoring_Overview`** | `Core` | Page | `ITAdmin` | Diagnostic console tracking in-browser SQLite storage size, sync queue health, and audit logs. |

---

## 6. Step-by-Step Developer Troubleshooting Playbooks

---

### Playbook A: Adding a New Question Tag to an Inspection Template
1. **Identify the Semantic `tagId`**: Determine a unique, lowercase, underscore-delimited tag name (e.g. `val_ups_inverter_temp_phase_a`). Never reuse an existing tag ID.
2. **Define in Configuration**:
   * Open `DynamicQuestionnaire.QuestionConfig_Overview` in the Admin Console.
   * Create a new `QuestionConfig` record, assigning `TagId = val_ups_inverter_temp_phase_a`, Label = "Inverter Phase A Temperature", and `QuestionType = Decimal`.
3. **Assign to Page**:
   * Open the target `PageConfig` and add the `QuestionConfig` to the desired row order.
4. **Define Rules (Optional)**:
   * If the question has mathematical limits (e.g., must be between $20^\circ\text{C}$ and $95^\circ\text{C}$), create a `ValidationRule` with formula `val_ups_inverter_temp_phase_a >= 20 and val_ups_inverter_temp_phase_a <= 95`.
5. **Verify Local Clone**:
   * In local development, create a test job (`Core.ACT_CreateJob_Open_Device`) and verify that the runtime `Question` entity is instantiated with the proper `tagId` and data type.

---

### Playbook B: Registering and Dispatching a New Tag Action
1. **Add Enum Value (If New Action Type)**:
   * In Studio Pro, open enumeration `DynamicQuestionnaire.TagActionType` and add the new action literal (e.g. `CalculateCoolingDemand`).
2. **Implement the Action Nanoflow**:
   * In the target module (e.g. `Thermal`), create nanoflow `SUB_CalculateCoolingDemand` accepting `ImpactingQuestion`, `Questionnaire`, `Job`, `CurrentPage`, `PageListName`.
3. **Register in Dispatcher**:
   * Open nanoflow `DynamicQuestionnaire.SUB_QuestionBehavior_Dispatcher`.
   * In the exclusive split `ActionType`, add a case for `CalculateCoolingDemand` routing to your new nanoflow.
4. **Configure Database Record**:
   * Open `DynamicQuestionnaire.TagActionConfig_Overview`.
   * Add a row: `ImpactingTagId = 'val_ambient_return_air_temp'`, `Technology = Thermal`, `ActionType = CalculateCoolingDemand`, `ExecutionOrder = 1`.
5. **Test Reactive Execution**:
   * Open a Thermal inspection on the mobile client, change `val_ambient_return_air_temp`, and observe the automatic invocation of `SUB_CalculateCoolingDemand`.

---

### Playbook C: Debugging Offline SQL Queries & Formula Evaluator
1. **Enable Browser Debugging Cookies**:
   * Open browser Developer Tools (F12) $\rightarrow$ Application / Storage $\rightarrow$ Cookies.
   * Add cookie: `debugFormulaEngine = true`.
   * Add cookie: `debugCalcRules_Impacting = true` or `debugCustomTagBehaviour = true`.
2. **Trigger the Change**:
   * Modify an input on the questionnaire page.
   * The browser will automatically hit the `debugger;` statement inside `JSA_CustomTagBehaviour_RunByImpacting.js` or `JSA_CalculateRules_RunByImpacting.js`.
3. **Inspect the Local SQL Statement**:
   * In the browser Console, inspect the `sql` string variable.
   * Verify that `mx${userGUID}$DynamicQuestionnaire$...` matches the active user GUID.
   * Execute the query directly in the Web SQL / SQLite developer console to verify returned row counts.

---

### Playbook D: Testing NextPredict Excel / Hioki Ingestion Locally
1. **Prepare Test Spreadsheet**:
   * Ensure column 1 contains `String Number` (integer), column 2 contains `Jar Number` (integer), column 3 contains `Float Voltage` (decimal), and column 4 contains `Internal Resistance` (decimal).
2. **Generate Matrix First**:
   * On the Battery configuration screen, set string count to 2 and jar count to 40.
   * Confirm that `SUBOCH_Battery_MeasuresData_Generate` created 80 `BatteryMeasure` rows in the local database.
3. **Upload File**:
   * Navigate to `SNIP_UploadBatteryMeasures`, select file type `NextPredict Standard Excel`, and drop the file.
4. **Inspect Validation Output**:
   * If any cells violate tolerances, inspect `Core.ErrorCollection` DataGrid to view exact line numbers and rejection reasons.
   * Switch to `SNIP_MeasuresChart` to confirm that the Plotly bar chart rendered voltage and resistance profiles correctly.
