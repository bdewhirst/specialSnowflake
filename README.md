# specialSnowflake
quick repo to record learnings from Snowflake tutorials, etc.

#### Brian Dewhirst | b.dewhirst@gmail.com

**please note** while this is public for visibility, yes, this is just someone running thru a SaaS vendor's tutorial.

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

# first tutorial script (complete; 'zero to snowflake' tutorial)

```create or replace table trips
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

-- this 'zero copy cloning' is really cool
create table trips_dev clone trips;  -- doesn't double the storage requirements

-- beginning of step 7 of tutorial; weather analysis and working with semi-structured data
create or replace database weather;  -- my edit, making it re-entrant (at the risk of blowing away the new database)

use role accountadmin;  -- access control hasn't been covered, and I've tweaked this step to not be sysadmin to align w/ my related workaround.
use warehouse compute_wh;
use database weather;
use schema public;

create table json_weather_data (v variant);  -- snowflake has a special column type (VARIANT) that allows storing the entire json as a single row
-- they call this "semi-structured data magic"

create stage nyc_weather
url = 's3://snowflake-workshop-lab/zero-weather-nyc';

list @nyc_weather;

copy into json_weather_data
from @nyc_weather 
    file_format = (type = json strip_outer_array = true);

select * from json_weather_data limit 10;  -- each row is a big json; the table has one column called "V"

-- note: snowflake supports multiple comment syntaxes
// create a view that will put structure onto the semi-structured data
create or replace view json_weather_data_view as
select
    v:obsTime::timestamp as observation_time,
    v:station::string as station_id,
    v:name::string as city_name,
    v:country::string as country,
    v:latitude::float as city_lat,
    v:longitude::float as city_lon,
    v:weatherCondition::string as weather_conditions,
    v:coco::int as weather_conditions_code,
    v:temp::float as temp,
    v:prcp::float as rain,
    v:tsun::float as tsun,
    v:wdir::float as wind_dir,
    v:wspd::float as wind_speed,
    v:dwpt::float as dew_point,
    v:rhum::float as relative_humidity,
    v:pres::float as pressure
from
    json_weather_data
where
    station_id = '72502';

select * from json_weather_data_view
where date_trunc('month',observation_time) = '2018-01-01'
limit 20;

-- here we join data notionally in our analytics environment (citibike trips) with json weather data (thru a view that lets us treat fields as columns)
select 
 weather_conditions as conditions
,count(*) as num_trips
from citibike.public.trips
left outer join json_weather_data_view
on date_trunc('hour', observation_time) = date_trunc('hour', starttime)
where conditions is not null
group by 1 order by 2 desc;


drop table json_weather_data;
select * from json_weather_data limit 10;  -- sic; we're proving the drop worked.

undrop table json_weather_data;

--verify table is undropped
select * from json_weather_data limit 10;

-- step 8; using time travel
use role accountadmin;  -- access control hasn't been covered, and I've tweaked this step to not be sysadmin to align w/ my related workaround.
use warehouse compute_wh;
use database citibike;
use schema public;

update trips set start_station_name = 'oops'; -- deliberate introduction of a data-changing error

select
start_station_name as "station",
count(*) as "rides"
from trips
group by 1
order by 2 desc
limit 20;


-- run command to find the query id of the last update command and store it in a variable named $QUERY_ID

set query_id =
(select query_id from table(information_schema.query_history_by_session (result_limit=>5))
where query_text like 'update%' order by start_time desc limit 1);

create or replace table trips as
(select * from trips before (statement => $query_id));

select
start_station_name as "station",
count(*) as "rides"
from trips
group by 1
order by 2 desc
limit 20;

-- step 9: working with roles, account admin, and account usage-- maybe this is where I figure out why I've had to tweak a few things...
-- ACCOUNTADMIN is both SYSADMIN and SECURITYADMIN system-defined roles; so this is running with elevated permissions; fine for practice, dangerous in practice.
create role junior_dba;
-- grant role junior_dba to user --... "[MY USERNAME HERE]"; -- run once w/ username in place.

use role junior_dba;

use role accountadmin;

grant usage on warehouse compute_wh to role junior_dba;
use role junior_dba;
use warehouse compute_wh;

use role accountadmin;
grant usage on database citibike to role junior_dba;
grant usage on database weather to role junior_dba;

use role junior_dba;

-- n.b.:  note that access in-script and access in the web UI are separate!



-- step 10: sharing data securely & the data marketplace
-- (this is handled in the UI, but note that there are free sources of data in the marketplace)


-- step 11: resetting your snowflake environment
-- (post-lab clean-up)
use role accountadmin;

drop share if exists zero_to_snowflake_shared_data;
-- If necessary, replace "zero_to_snowflake-shared_data" with the name you used for the share

drop database if exists citibike;
drop database if exists weather;
drop warehouse if exists analytics_wh;
drop role if exists junior_dba;

```
