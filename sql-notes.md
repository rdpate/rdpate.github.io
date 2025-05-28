---
title: SQL Notes
---

## Links
* [Oracle SQL Functions](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/Functions.html#GUID-D079EFD3-C683-441F-977E-2C9503089982)
* [Oracle Database 19c - Get Started](https://docs.oracle.com/en/database/oracle/oracle-database/19/index.html)
* MS SQL Server
    * [Functions](https://learn.microsoft.com/en-us/sql/t-sql/functions/functions?view=azuresqldb-current)
        * [generate_series](https://learn.microsoft.com/en-us/sql/t-sql/functions/generate-series-transact-sql?view=azuresqldb-current)
        * [datetrunc, date_bucket, ...](https://learn.microsoft.com/en-us/sql/t-sql/functions/date-and-time-data-types-and-functions-transact-sql?view=azuresqldb-current)
    * [Date and Time Types](https://learn.microsoft.com/en-us/sql/t-sql/data-types/date-and-time-types?view=azuresqldb-current)
        * Documentation recommends use only: date, time, datetime2, datetimeoffset
        * However, datetimeoffset is not an offset between two datetime2 values, but is instead a datetime2 plus a timezone offset.  Our EDP does not currently store timezones, and we likely gain no benefit from doing so until it does.
    * [Merge](https://sqlblog.org/merge)
        * Use "merge TARGET with (holdlock)".  ([source](https://sqlblog.org/merge))
        * Don't merge on temporal tables if the history table has a nonclustered index. ([source](https://sqlserverfast.com/blog/hugo/2023/09/an-update-on-merge/), issue #8)
        * Don't merge with both update and delete actions without also having an insert action. ([source](https://sqlserverfast.com/blog/hugo/2023/09/an-update-on-merge/), issue #9)
        * Don't merge on a partitioned table with only one partition (which shouldn't occur). ([source](https://sqlserverfast.com/blog/hugo/2023/09/an-update-on-merge/), issue #6)
    * [Regular Expressions](https://learn.microsoft.com/en-us/sql/t-sql/functions/regular-expressions-functions-transact-sql?view=azuresqldb-current)
        * [RE2 regex syntax](https://github.com/google/re2/wiki/Syntax)
* [SQLines: Oracle to Microsoft SQL Server (MSSQL) Migration](https://www.sqlines.com/oracle-to-sql-server/)
    * [SQLines: TRUNC(datetime)](https://www.sqlines.com/oracle-to-sql-server/trunc_datetime)
* [SQL Dialects Reference (Wikibooks)](https://en.wikibooks.org/wiki/SQL_Dialects_Reference)
* [What do BETWEEN and the devil have in common?](https://sqlblog.org/2011/10/19/what-do-between-and-the-devil-have-in-common)

# Azure SQL Server

## List of Numbers

    -- generate_series(start, stop[, step])
    --  Stop is included in the output.
    --  Step defaults to 1 if start <= stop, or -1 otherwise.  Step cannot be 0.
    --  No output if (start < stop and step < 0) or (stop < start and step > 0).
    select value from generate_series(0, 42)

## Dates, Times, & Timestamps

    -- Use a [begin, end) range when the end is NOT included in the range.
    -- This range is empty when end <= begin, and convention is the empty range uses begin = end.
    -- This is useful to define periods by their start and the next period's start, such as [Sunday, next Sunday).
    declare @begin date = '2025-01-01'
    declare @end   date = '2025-03-03'
    
    -- Use a [first, last] range when the last IS included in the range.
    -- This range is empty when last < first.
    declare @first date = '2025-01-01'
    declare @last  date = '2025-03-02'
    
    -- Change date to start (Sunday) of same week.
    -- (This is short enough that these variables aren't used below.)
    declare @begin_sunday date = datetrunc(week, @begin)
    declare @first_sunday date = datetrunc(week, @first)
    
    -- Change date to last day (Saturday) in same week.
    declare @end_bwe   date = dateadd(day, 6, datetrunc(week, @end))
    declare @last_bwe  date = dateadd(day, 6, datetrunc(week, @last))
    
    select @begin as "begin", @end as "end", @end_bwe  as end_bwe
         , @first as first  , @last as last, @last_bwe as last_bwe
         
    
    -- Dates between @begin and @end (exclusive).
    select dateadd(day, value, @begin) as date
    from generate_series(0, datediff(day, @begin, @end) - 1, 1)
    
    -- Dates between @first to @last (inclusive).
    select dateadd(day, value, @first) as date
    from generate_series(0, datediff(day, @first, @last), 1)
    
    
    -- Saturdays between @begin and @end (exclusive).
    -- (This makes more sense with Saturday dates than with arbitrary dates as we have here.)
    select dateadd(day, 6 + 7 * value, datetrunc(week, @begin)) as bwe
    from generate_series(0, datediff(week, @begin, dateadd(week, -1, @end)), 1)
    
    -- Saturdays between @first to @last (inclusive).
    select dateadd(day, 6 + 7 * value, datetrunc(week, @first)) as bwe
    from generate_series(0, datediff(week, @first, @last), 1)


# Oracle

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

    WITH report_windows AS (
        -- purpose: Return [begin, end] for each month from first_year and first_month_no through today, excluding months not between first_month_no and last_month_no (which possibly wraps around New Year's Day).
        SELECT window_begin
             , add_months(window_begin, 1) - ( 1 / 60 / 60 / 24 ) AS window_end
        FROM (
            SELECT add_months(first_month, level - 1) AS window_begin
                 , first_month_no
                 , last_month_no
            FROM (
                SELECT x.*
                     , add_months(TO_DATE(first_year || '-01-01', 'YYYY-MM-DD')
                                  , first_month_no - 1) AS first_month
                FROM (
                    SELECT 2015 AS first_year       -- First year included.
                         , 10   AS first_month_no   -- First month included.
                         , 3    AS last_month_no    -- Last month included.  If less than first_month, then in the next year.  If equal to first_month, then only one month per year.
                    FROM dual
                ) x
            )
            CONNECT BY
                level <= months_between(add_months(trunc(sysdate, 'year')
                                                 , 11)
                                      , first_month)
        )
        WHERE window_begin <= sysdate
              AND mod(EXTRACT(MONTH FROM window_begin) + 12 - first_month_no
                    , 12) <= mod(last_month_no + 12 - first_month_no, 12)
    )
    SELECT *
    FROM report_months;

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
