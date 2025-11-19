# Snowflake pipeline

Short description
- End-to-end Snowflake pipeline that ingests JSON log files from an S3 stage, persists raw records, enriches them (IP geolocation + local time/TOD), and incrementally upserts deduplicated enriched events using Snowpipe, Streams, and Tasks.

Contents
- Overview
- Architecture & components
- Quick start (prerequisites + basic steps)
- Key SQL objects created
- Operational notes & checks
- Troubleshooting checklist
- Recommended enhancements
- Change log / author

Overview
- Ingest JSON logs from S3 (array- or newline-wrapped JSON).
- Persist parsed fields and ingestion metadata in RAW.ED_PIPELINE_LOGS.
- Maintain a TIME_OF_DAY lookup for human-friendly TOD labels.
- Enrich raw events with geolocation (IP → city/region/country/timezone) and convert UTC timestamps to local time.
- Use a CDC stream and a scheduled task to MERGE new enriched events into ENHANCED.LOGS_ENHANCED.

Architecture & components
- External stage: RAW.UNI_KISHORE_PIPELINE (S3, directory listing enabled).
- File format: RAW.FF_JSON_LOGS (JSON, strip_outer_array = TRUE).
- Raw table: RAW.ED_PIPELINE_LOGS (parsed fields + METADATA$FILENAME, METADATA$FILE_ROW_NUMBER, load timestamp).
- Snowpipe: RAW.PIPE_GET_NEW_FILES (auto_ingest + SNS topic).
- Time-of-day lookup: RAW.TIME_OF_DAY_LU.
- CDC stream: RAW.ED_CDC_STREAM on RAW.ED_PIPELINE_LOGS.
- View: RAW.LOGS for consistent parsed access.
- Enhanced table: ENHANCED.LOGS_ENHANCED (joins to IPINFO_GEOLOC demo.location).
- Task: RAW.CDC_LOAD_LOGS_ENHANCED (scheduled MERGE, runs when stream has data).

Quick start / prerequisites
- Snowflake account with privileges to:
    - CREATE DATABASE, SCHEMA, PIPE, TASK, STREAM, TABLE, FILE FORMAT, STAGE.
    - Register and configure storage integration or supply credentials for the S3 stage.
- SNS topic and S3 notifications configured for Snowpipe auto-ingest (if using auto_ingest).
- IP geolocation dataset and helper functions (IPINFO_GEOLOC schema with TO_JOIN_KEY, TO_INT, demo.location).
- Recommended role for provisioning: SYSADMIN (use least-privilege roles for production runs).

Basic setup steps (run in a Snowflake worksheet)
1. Set role and create database/schema:
     - USE ROLE SYSADMIN;
     - CREATE DATABASE AGS_GAME_AUDIENCE;
2. Create stage:
     - CREATE STAGE RAW.UNI_KISHORE_PIPELINE URL='s3://uni-kishore-pipeline' DIRECTORY=(ENABLE=TRUE);
3. Create file format:
     - CREATE FILE FORMAT RAW.FF_JSON_LOGS TYPE=JSON STRIP_OUTER_ARRAY=TRUE;
4. Create raw table (or run the provided CREATE TABLE AS SELECT).
5. Create and resume the pipe (set aws_sns_topic to your SNS ARN).
6. Populate TIME_OF_DAY_LU.
7. Create stream, view, enhanced schema/table, and task; resume task.

Key SQL object notes
- COPY INTO file_format must match the FILE FORMAT name exactly.
- RAW.ED_PIPELINE_LOGS uses metadata columns (METADATA$FILENAME, METADATA$FILE_ROW_NUMBER) for provenance.
- ENHANCED.LOGS_ENHANCED relies on IPINFO_GEOLOC functions/tables — ensure join logic and timezone strings are valid.
- Task MERGE keys: GAMER_NAME + GAME_EVENT_UTC + GAME_EVENT_NAME; consider adding file+row metadata if duplicates are possible.

Operational considerations
- Data quality: Use ON_ERROR, TRY_CAST or validation_mode during rollout to catch malformed JSON and partial failures.
- Idempotency: MERGE keys should uniquely identify events; consider including file/row metadata or hashes for stronger dedup.
- Cost & performance: Monitor task frequency and warehouse size; consider clustering or denormalization for large geolocation joins.
- Observability: Monitor PIPE_STATUS, PIPE_LOAD_HISTORY, TASK_HISTORY, and SYSTEM$STREAM_HAS_DATA.

Troubleshooting checklist
- Stage access: run LIST @<stage> to verify Snowflake can list files.
- File parsing: run SELECT $1 FROM @stage (FILE_FORMAT => '...') to preview parsing.
- Snowpipe: check pipe load history and verify SNS configuration.
- Stream: SHOW STREAMS; SELECT PARSE_JSON(SYSTEM$STREAM_HAS_DATA('ed_cdc_stream')).
- Task failures: inspect TASK_HISTORY and QUERY_HISTORY; increase warehouse size or adjust schedule if timeouts occur.

Recommended enhancements
- Add ingestion error table to capture rejected rows or copy diagnostics.
- Use validation_mode='RETURN_ERRORS' during initial rollout.
- Compute row-level hash/checksum for reliable dedup/updates.
- Add masking/row access policies for PII (user_login, ip_address).
- Automate tests (timezone conversion, IP geolocation join correctness).
- Consider search optimization or precomputed lookup for high-throughput IP joins.

Security & governance
- Use least-privilege roles for runtime and an approval process for object creation.
- Protect PII via masking, access controls, and retention policies.
- Secure S3 + SNS with appropriate IAM and storage integration configuration.

Change log / author
- Version: 1.0
- Author: pipeline author (add name & date when promoting to production)
- Recommended: record schema and logic changes in subsequent versions.

Files included (SQL)
- Pipeline SQL script that creates: database, schemas, stage, file format, raw table, pipe, time_of_day_lu, stream, view, enhanced schema/table, and scheduled MERGE task.

How to run safely
- Run in a dev worksheet with small sample files first.
- Use RETURN_ERRORS or a small test file set; verify outputs (raw rows, enriched rows) before enabling production auto-ingest and task resume.

Contact / ownership
- Add the pipeline owner's email, on-call rotation, and runbook location here before production release.
