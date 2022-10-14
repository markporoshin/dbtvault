## Example

Filter source and target by period

Following model

```SQL

{{
    config(
        ...
        materialized='vault_insert_by_period',
        timestamp_field='load_date',
        start_date='2022-01-02 00:00:00',
        stop_date='2022-01-03 00:00:00'
    )
}}


{%- set source_model = '<stage_model>' -%}
{%- set src_pk = "<primary_key>" -%}
{%- set src_nk = "<business_key>" -%}
{%- set src_ldts = "load_date" -%}
{%- set src_source = "record_source" -%}


{{ dbtvault.hub(src_pk=src_pk, src_nk=src_nk, src_ldts=src_ldts,
                src_source=src_source, source_model=source_model) }}

```

will produce

```SQL

WITH row_rank_1 AS (
    SELECT * FROM (
        SELECT rr.source_pk, rr.source, rr.load_date, rr.record_source,
               ROW_NUMBER() OVER(
                   PARTITION BY rr.source_pk
                   ORDER BY rr.load_date
               ) AS row_number
        FROM "<source>" AS rr
        WHERE rr.source_pk IS NOT NULL
    ) h
    WHERE row_number = 1
),

stage_mat_filter AS (
    SELECT *
    FROM row_rank_1
    WHERE DATE_TRUNC(
            'day', '2022-01-02 00:00:00'::TIMESTAMP + INTERVAL '0 day') <= load_date::TIMESTAMP
            AND
            load_date::TIMESTAMP < DATE_TRUNC('day', '2022-01-03 00:00:00'::TIMESTAMP + INTERVAL '0 day')
),
records_to_insert AS (
    SELECT a.source_pk, a.source, a.load_date, a.record_source
    FROM stage_mat_filter AS a
    LEFT JOIN (
        SELECT
            source_pk
        FROM {{ this }}
        WHERE
            DATE_TRUNC('day', '2022-01-02 00:00:00'::TIMESTAMP + INTERVAL '0 day') <= load_date::TIMESTAMP
            AND
            load_date::TIMESTAMP < DATE_TRUNC('day', '2022-01-03 00:00:00'::TIMESTAMP + INTERVAL '0 day')
    ) AS d
    ON a.source_pk = d.source_pk
    WHERE d.source_pk IS NULL
)

SELECT * FROM records_to_insert

```
