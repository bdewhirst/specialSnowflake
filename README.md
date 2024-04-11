# specialSnowflake
quick repo to record learnings from Snowflake tutorials, etc.

#### Brian Dewhirst | b.dewhirst@gmail.com

# Starting from here:
## Ref: https://quickstarts.snowflake.com/guide/getting_started_with_snowflake/index.html?index=..%2F..index#0

(I selected the 'standard' edition, GCP as my cloud provider, and indicated my interests)

(see local notes for username and password)

30 day free trial started 2024-04-10 at approx. 12:00 EDT


# transferable concepts

- Much like MySQL, the "user admin" (etc.) can be handled inside the tool. (check the 'Admin' tab)

- the web gui accepts "control+enter" as the command to execute a highlighted bit of SQL

- you can scale resources up or down easily

- it caches intelligently (and doesn't charge compute for re-running a cached query).

- "Snowflake to MySQL" is a pattern to consider:  https://www.basedash.com/blog/snowflake-to-mysql-a-guide

### misc notes WRT first tutorial
- screenshots are slightly out of date (as noted)
- although demo/tutorial says to use sysadmin, I have been using accountadmin; this is likely permissions related (but may also reflect changing security recommendations for Snowflake.)

# first tutorial script

```
create or replace table trips
(tripduration integer,
starttime timestamp,
stoptime timestamp,
start_station_id integer,
start_station_name string,
start_station_latitude float,
start_station_longitude float,
end_station_id integer,
end_station_name string,
end_station_latitude float,
end_station_longitude float,
bikeid integer,
membership_type string,
usertype string,
birth_year integer,
gender integer);


list @citibike_trips;

--create file format

create or replace file format csv type='csv'
  compression = 'auto' field_delimiter = ',' record_delimiter = '\n'
  skip_header = 0 field_optionally_enclosed_by = '\042' trim_space = false
  error_on_column_count_mismatch = false escape = 'none' escape_unenclosed_field = '\134'
  date_format = 'auto' timestamp_format = 'auto' null_if = ('') comment = 'file format for ingesting data for zero to snowflake';

  -- verify file format is created
  show file formats in database citibike;

  copy into trips from @citibike_trips file_format=csv PATTERN = '.*csv.*' ;

-- clear out data to re-try w/ larger engine (tutorial step )
truncate table trips;

--verify table is clear
select * from trips limit 10;

--load data with large warehouse
show warehouses;

copy into trips from @citibike_trips
file_format=CSV;
  
select * from trips limit 20;

-- here's where we (seemlessly) changed to the new analytics warehouse we created with a few clicks

select date_trunc('hour', starttime) as "date",
count(*) as "num trips",
avg(tripduration)/60 as "avg duration (mins)",
avg(haversine(start_station_latitude, start_station_longitude, end_station_latitude, end_station_longitude)) as "avg distance (km)"
from trips
group by 1 order by 1;


select
monthname(starttime) as "month",
count(*) as "num trips"
from trips
group by 1 order by 2 desc;

```
