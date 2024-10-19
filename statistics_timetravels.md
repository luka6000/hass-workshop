# Today's statistics for yesterday's readings

I have a few sensors for remote meters (hot and cold water, heat). These meters provide new values once per day, typically sometime after midnight. They populate statistics for the 0am or 1am hour. The problem is that after aggregation, this water/heat consumption is recorded a full day later than when it actually happened.

# Solution: statistics time travels
I'm using this query to adjust the statistics after the last meter update, moving the current day's statistics back to 11pm on the previous day. You need to wait for the statistics to be calculated after the next full hour following the meter update.

```
update statistics as s set state=t.state, sum=t.sum
from (
    select metadata_id, max(state) as state, max(sum) as sum, unixepoch('now','-1 day','start of day','+23 hours')-unixepoch('now','localtime')+unixepoch('now') as start_ts
    FROM "statistics"
    where
        metadata_id in (select id from statistics_meta where statistic_id in (
        'sensor.cold_water_meter', 'sensor.hot_water_meter', 'sensor.cwu_meter', 'sensor.heat_meter'
        ))
        and created_ts> unixepoch('now','start of day')-unixepoch('now','localtime')+unixepoch('now') 
        and created_ts< unixepoch('now','start of day','+23 hours','+59 minutes')-unixepoch('now','localtime')+unixepoch('now')
    group by metadata_id
) t
where s.metadata_id=t.metadata_id and s.start_ts>=t.start_ts
;
```
This query is idempotent. It means it will not break once the statistics are moved, no matter how many times you run it on the current day.\
It can also be used to aggregate all daily updates into a single value, moved to 11pm yesterday. Or any other hour once you modify it.
# Explanation
```
update statistics as s set state=t.state, sum=t.sum
```
Adjust _state_ and _sum_ statistic values with

```
select metadata_id, max(state) as state, max(sum) as sum, unixepoch('now','-1 day','start of day','+23 hours')-unixepoch('now','localtime')+unixepoch('now') as start_ts
```
max values already recorded today for state and sum. Additionally calculate UTC 11pm timestamp using local timezone difference.

```
FROM "statistics"
where
    metadata_id in (select id from statistics_meta where statistic_id in (
    'sensor.cold_water_meter', 'sensor.hot_water_meter', 'sensor.cwu_meter', 'sensor.heat_meter'
    ))
```
These are my meters entity_ids.

```
    and created_ts> unixepoch('now','start of day')-unixepoch('now','localtime')+unixepoch('now') 
    and created_ts< unixepoch('now','start of day','+23 hours','+59 minutes')-unixepoch('now','localtime')+unixepoch('now')
group by metadata_id
```
Use current day for max aggregations. Use local timezone difference on timestamps.

```
where s.metadata_id=t.metadata_id and s.start_ts>=t.start_ts
```
Update statistics for used entity_ids starting from calculated timestamp for yesterday 11pm.

# Automate it
You'll need a shell script.\
It will install the sqlite3 package to the Home Assistant image, but will do so only the first time it's used.
```
file="/usr/bin/sqlite3"
if [ -f "$file" ]
then
   echo "sqlite $file found."
else
   echo "Re-Installing sqlite:"
   apk add --update sqlite
fi

echo "update statistics as s set state=t.state, sum=t.sum
      from (
          select metadata_id, max(state) as state, max(sum) as sum, unixepoch('now','-1 day','start of day','+23 hours')-unixepoch('now','localtime')+unixepoch('now') as start_ts
          FROM "statistics"
          where
              metadata_id in (select id from statistics_meta where statistic_id in (
              'sensor.cold_water_meter', 'sensor.hot_water_meter', 'sensor.cwu_meter', 'sensor.heat_meter'
              ))
              and created_ts> unixepoch('now','start of day')-unixepoch('now','localtime')+unixepoch('now') 
              and created_ts< unixepoch('now','start of day','+23 hours','+59 minutes')-unixepoch('now','localtime')+unixepoch('now')
          group by metadata_id
      ) t
      where s.metadata_id=t.metadata_id and s.start_ts>=t.start_ts
      ;" |sqlite3 /config/home-assistant_v2.db
```
Make it run 1h after meter update and you'll be fine.\
That's it. Statistics time travels.
