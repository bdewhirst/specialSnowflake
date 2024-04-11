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

...
```
