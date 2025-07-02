# Edgy output validation

## Overview

This repository contains Databricks notebooks for evaluating the correctness of [Edgy](https://github.com/squareup/java/tree/master/edgy)'s output events. The goal of the validation is to ensure Edgy reliably produces correct CashConnectedGraphChangeEvent outputs for Cash account holders.

**Key Approach:**  
Validation is performed by calculating the *expected set of output events* using the **Snowflake tables** that mirror Duplograph's authoritative DynamoDB `edges` and `label_events` tables. These derived expected results are then rigorously compared to the actual events produced by Edgy, enabling automated detection and classification of any mismatches, gaps, or errors.

## Validation Methodology

1. **Choice of input events to validate:**  
   Backfill validation used a test dataset of a few thousand account holders that were hand chosen by modellers and backfilled by Edgy.
   BAU validation used a randomly selected set of edge/label events that occurred within a particular day of captured Edgy output events.

2. **Edgy output loading:**  
   The actual KnowledgeEvents produced by Edgy during the relevant validation period were loaded into a Databricks table from the S3 archive using the `edgy_s3_output_to_databricks_table.ipynb` notebook.

3. **Event & Label State Calculation:**  
   For event to be validated (account holder, edge, or label event), the notebooks calculate the *expected* output events based on the data available in the `duplograph.public.edges` and `duplograph.public.label_events` tables at the `effective_at` time of the event (if it is a BAU event) or at the time that Edgy ran the backfill if it is a backfilled event.

4. **Automated Comparison:**  
   Each notebook compares actual vs. expected events, categorizing and reporting:
   - Missing events (expected but not present in Edgy's output)
   - Unexpected events (not expected per graph state)
   - Property mismatches (timestamp, label propagation, connection types, etc)
   - Duplicates or inconsistent event IDs, and other data integrity issues

5. **Summary & Debugging:**  
   - Detailed pass/fail logs and summaries are generated for each subject
   - Analysis cells and exported CSVs allow for root-cause deep-dives

## Notebook Summaries

### 1. **`edgy_backfill_validation_20250515_final.ipynb`**

**Purpose:**  
Performed validation for *backfill* event production by Edgy based on a hand-chosen set of backfilled account holders.

**Core actions:**
- For each account holder, reconstructs their expected connected graph at a historical point-in-time using Snowflake query logic.
- Validates that all expected CashConnectedGraphChangeEvents were produced by Edgy for those historical connections, with correct event IDs, labels, and timestamps.
- Reports any missing/unexpected events or inconsistencies, as well as aggregate statistics.
- Contains summaries of manual investigations undertaken for the validation errors that were identified.

### 2. **`edgy_bau_edge_validation.ipynb`**

**Purpose:**  
Validates Edgy's live processing of BAU Edge events from Duplograph to CashConnectedGraphChangeEvent outputs.

**Core actions:**
- Loads a sample of live (BAU) Edge events that occurred within a period covered by available Edgy output data.
- For each, computes the expected ConnectionChange events based on the Snowflake graph state at the `effective_at` timestamp of the Edge event.
- Checks that expected and actual events match in presence, event ID convention, deduplication, timestamps, and labels.
- Performs general output sanity checks.

### 3. **`edgy_bau_label_validation.ipynb`**

**Purpose:**  
Validates Edgy’s BAU label change event output, focusing on correctness of label-driven events.

**Core actions:**
- Loads a sample of live (BAU) LabelEvents that occurred within a period covered by available Edgy output data.
- Computes the expected LabelChange events based on the state of the graph in the Snowflake tables at the `effective_at` timeststamp of the LabelEvent.
- Verifies that Edgy output matches the expected event set, and that connection types and event timestamps are correct.
- Performs general output sanity checks.

### 4. **`edgy_s3_output_to_databricks_table.ipynb`** 

**Purpose:**  
Loads Edgy's output data from avro files in S3 into a Databricks table so it can be queried by the validation notebooks.

**Important:**  
This notebook does not perform validation itself. Instead, it provides table ingestion and data preparation utilities so the actual validator notebooks can efficiently read Edgy event data alongside expected events from Snowflake.

## How to Use This Repository

1. In your Databricks workspace click "Create" -> "Git folder" and enter this repository's URL. This will checkout the notebooks into an `edgy-validation` git folder in your workspace and provide version control capabilities.
2. Run `edgy_s3_output_to_databricks_table.ipynb` (as needed) to load Edgy outputs data into a Databricks table.
3. Choose the appropriate validation notebook for your workflow (backfill, BAU edge, BAU label).
4. Follow the instructions in each notebook to run the validation. The parameters you need to update to run the validation over a different period can be found in the `Validation execution` section.
5. Explore and debug failures using built-in summary tables.

## Further Notes

- The validation fundamentally relies on **Snowflake as the source of truth** for Duplograph's graph state, specifically the `duplograph.public.edges` and `duplograph.public.label_events` tables. Sometimes the data in these tables doesn't match exactly with what's in Duplograph's DynamoDB tables, and this can result in some amount of validation errors.
- All validation logic is fully transparent (see notebook Markdown for SQL queries and rationale).
- All notebooks output both summary data and row-level CSVs for auditability.

## Further Reading

- [Duplograph SAM Design Doc](https://docs.google.com/document/d/16EYmWQz68lurF_CWh5RlLcL8gQiImZWWtcF_JgURWzE/edit?tab=t.0#heading=h.uinirtej7s73): explains Edgy's processing logic
- [Duplograph SAM Backfill](https://docs.google.com/document/d/1Mz1dPsUDQr76eTijlEz53VCDDk5DUY0RZtuk370RJ9A/edit?tab=t.0#heading=h.x6ryaflb29e7) explains the backfill logic

## Questions?

Reach out in #proj-edgy
