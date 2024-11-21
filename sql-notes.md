---
title: SQL Notes
---

## Links
* [Oracle SQL Functions](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Functions.html#GUID-D079EFD3-C683-441F-977E-2C9503089982)
* [Oracle Database 19c - Get Started](https://docs.oracle.com/en/database/oracle/oracle-database/19/index.html)
* [MS SQL Server Database Functions](https://learn.microsoft.com/en-us/sql/t-sql/functions/functions)

## Dates, Times, & Timestamps

    ALTER SESSION SET NLS_DATE_FORMAT = 'YYYY-MM-DD';  -- or 'YYYY-MON-DD'

<!-- -->

    ALTER SESSION SET NLS_DATE_FORMAT = 'YYYY-MM-DD HH24:MI:SS';  -- or 'YYYY-MON-DD HH:MI:SS PM'

<!-- -->

    WITH report_dates AS (
        SELECT begin_date + ( level - 1 ) AS report_date
        FROM (
            SELECT trunc(sysdate, 'd') - 21 AS begin_date
                 , trunc(sysdate)           AS end_date  -- This is inclusive and will be the last report_date.
            FROM dual
        )
        CONNECT BY
            level <= end_date + 1 - begin_date
    )
    -- Example:
    SELECT report_date
         , COUNT(some_date_column)  -- if using COUNT(*), then a window without any join matches will have COUNT(*) equal to 1, due to NULL
    FROM report_dates
    LEFT JOIN some_table_with_date_column ON ( some_date_column >= report_date AND some_date_column < report_date + 1 )
    GROUP BY report_date
    ORDER BY report_date;

<!-- -->
    
    WITH report_bwe AS (
        SELECT begin_sunday + ( level - 1 ) * 7 AS window_begin
             , begin_sunday + level * 7         AS next_window_begin
             , begin_sunday + level * 7 - 1     AS bwe_date
        FROM (
            SELECT trunc(sysdate - 7 * lookback_weeks, 'd')      AS begin_sunday
                 , trunc(sysdate + 7 * lookahead_weeks, 'd') - 1 AS end_saturday
            FROM (
                SELECT 8 AS lookback_weeks
                     , 0 AS lookahead_weeks  -- 0 excludes current week, 1 includes current week, 2 includes current and next week, etc.
                FROM dual
            )
        )
        WHERE trunc(begin_sunday, 'd') = begin_sunday          -- check that begin_sunday is a Sunday
              AND trunc(end_saturday, 'd') + 6 = end_saturday  -- check that end_saturday is a Saturday
              AND begin_sunday < end_saturday                  -- check that begin_sunday is before end_saturday
        CONNECT BY
            level <= (end_saturday + 1 - begin_sunday) / 7
    )
    SELECT bwe_date
         , COUNT(some_date_column)  -- if using COUNT(*), then a window without any join matches will have COUNT(*) equal to 1, due to NULL
    FROM report_bwe
    LEFT JOIN some_table_with_date_column ON ( some_date_column >= window_begin AND some_date_column < next_window_begin )
    GROUP BY bwe_date
    ORDER BY bwe_date;

<!-- -->
    
    WITH report_months AS (
        SELECT add_months(trunc(sysdate, 'month'), -(level - include_current))     AS month_begin
             , add_months(trunc(sysdate, 'month'), -(level - include_current) + 1) AS next_month_begin
        FROM (
            SELECT 3 AS completed_months  -- change to however many months to go back
                 , 1 AS include_current   -- change to 0 to exclude the current month
            FROM dual
        )
        CONNECT BY
            level <= completed_months + include_current
    )
    SELECT month_begin
         , COUNT(some_date_column)  -- if using COUNT(*), then a window without any join matches will have COUNT(*) equal to 1, due to NULL
    FROM report_months
    LEFT JOIN some_table_with_date_column ON ( some_date_column >= month_begin AND some_date_column < next_month_begin )
    GROUP BY month_begin
    ORDER BY month_begin;

<!-- -->
    
    WITH report_windows AS (
        SELECT begin_ts + ( level - 1 ) * duration AS window_begin
             , begin_ts + level * duration         AS next_window_begin
        FROM (
            SELECT trunc(sysdate, 'd') - 21 AS begin_ts   -- First window's beginning timestamp.
                 , trunc(sysdate)           AS end_ts     -- Last window's ending timestamp, but will be rounded up if the last window is not a full duration.
                 , 7                        AS duration   -- Window duration as fractions of a day.  2/24 is two hours; 7 is seven days.
    
                 -- to show year-plus history, these are convenient begin dates (to put in for begin_ts above)
                 , trunc(add_months(sysdate, -1), 'year')  AS ytd                 -- rolls over on Feb 1st
                 , trunc(add_months(sysdate, -13), 'year') AS last_year_plus_ytd  -- rolls over on Feb 1st
                 , trunc(add_months(sysdate, -25), 'year') AS two_years_plus_ytd  -- rolls over on Feb 1st
            FROM dual
        )
        CONNECT BY
            level <= ceil((end_ts - begin_ts) / duration)
    )
    SELECT window_begin
         , COUNT(some_date_column)  -- if using COUNT(*), then a window without any join matches will have COUNT(*) equal to 1, due to NULL
    FROM report_windows
    LEFT JOIN some_table_with_date_column ON ( some_date_column >= window_begin AND some_date_column < next_window_begin )
    GROUP BY window_begin
    ORDER BY window_begin;

<!-- -->

## Lists

These functions provide convenient inline lists.

### Levels
    
    WITH integers AS (
        SELECT level AS num
        FROM dual
        CONNECT BY
            level <= 42 -- number of levels, from 1 to this number (inclusive)
    )
    SELECT *
    FROM integers;

<!-- -->

### List of Numbers
    
    SELECT column_value
    FROM sys.odcinumberlist(1, 2, 3);

<!-- -->

### List of Strings
    
    SELECT column_value
    FROM sys.odcivarchar2list('a', 'b', 'c');

### List of Dates

    SELECT column_value
    FROM sys.odcidatelist(DATE '2022-10-21', DATE '2023-08-09', DATE '2024-04-19');

### Integer Buckets

* Last bucket is unbounded, but is written with an "impossibly large" upper limit to avoid checking NULL.

<!-- -->

    WITH buckets_minimums AS (
        SELECT ROWNUM       AS num
             , column_value AS min_value
        FROM sys.odcinumberlist(0, 31, 46, 61, 76
                              , 91, 121)
    ), buckets AS (
        SELECT CASE
                     WHEN max_value IS NULL THEN min_value || '+'
                     ELSE min_value
                              || '-'
                              || max_value
                 END                         AS name
             , min_value
             , coalesce(max_value, 999999999) AS max_value
        FROM (
            SELECT bucket.num
                 , bucket.min_value
                 , next_bucket.min_value - 1 AS max_value  -- NULL means unbounded
            FROM buckets_minimums bucket
            LEFT JOIN buckets_minimums next_bucket ON ( next_bucket.num = bucket.num + 1 )
        )
    )
    SELECT *
    FROM buckets
    ORDER BY max_value;

---

## Miscellaneous

Rounding a number to a number of significant digits; below rounds to 3 digits (subtract 1 first digit to get 2).

    -- Handles positive numbers:
    --  round(VALUE, -greatest(0, floor(log(10, VALUE)) - 2))
    -- Handles all real numbers:
    --  round(VALUE, -greatest(0, case when VALUE = 0 then 0 else floor(log(10, abs(VALUE))) - 2 end))
    
    SELECT value
         , round(value, - cut_places) AS rounded
         , log10
         , cut_places
    FROM (
        SELECT value
             , CASE
                     WHEN value != 0 THEN round(log(10
                                                      , abs(value))
                                                  , 4)
                 END AS log10
             , CASE
                     WHEN value = 0 THEN 0
                     ELSE floor(log(10
                                      , abs(value))) - 2
                 END AS cut_places
        FROM (
            SELECT column_value AS value
            FROM sys.odcinumberlist(- 12345.6789, - 1234.5678, - 123.4567, - 12.3456, - 1.2345
                                  , 0, 0.1234, 1.2345, 12.3456, 123.4567
                                  , 1234.5789, 12345.6789, 123456.789, 1234567.89, 12345678.9
                                  , 123456789, 1234567890)
        )
    );