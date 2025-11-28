# Queries

Here are a few queries to manipulate data associated to projects. These extend abilities of website with SQL client.

### Evaluate changes

Find features that get tags on a given version:

```sql
with lines_changes as (SELECT
    osmid,
    version,
    tags,
    lag(tags) OVER (PARTITION BY osmid ORDER BY version) AS tags_old,
    geom_len
  FROM pdm_features_lines_changes
  WHERE osmid LIKE 'w%' AND action != 'delete'
)
SELECT osmid, version, tags_old, tags, (not tags_old ? 'circuits' and tags ? 'circuits') as circuit_add
from lines_changes
where tags is not null and not tags_old ? 'circuits' and tags ? 'circuits'
```

Find features that get tags on a given version and on a given boundary

```sql
with lines_changes as (SELECT
    fc.osmid,
    fc.version,
    fc.tags,
    lag(tags) OVER (PARTITION BY fc.osmid ORDER BY fc.version) AS tags_old,
    fc.geom_len
  FROM pdm_features_lines_changes fc
  JOIN pdm_features_lines_boundary fb ON fb.osmid=fc.osmid AND fb.version=fc.version
  WHERE fc.osmid LIKE 'w%' AND fc.action != 'delete' AND fb.boundary=120027
)
SELECT osmid, version, tags_old, tags, (not tags_old ? 'circuits' and tags ? 'circuits') as circuit_add
from lines_changes
where tags is not null and not tags_old ? 'circuits' and tags ? 'circuits'
```

```sql
with nodes as (
    SELECT
    osmid,
    version,
    ts as ts_start,
    LEAD(ts) OVER (PARTITION BY osmid ORDER BY version) AS ts_end
    FROM pdm_features_supports
	WHERE osmid like 'n%' and action != 'delete' AND tags->>'power' IN ('tower','pole')
), list as (
	select f.osmid osmid, f.version version, n.osmid nid
	from pdm_features_lines_changes f
	join pdm_members_lines fm ON fm.osmid=f.osmid AND fm.version=f.version
	join nodes n ON n.osmid=fm.memberid AND ((greatest(f.ts_start, '2024-01-01') >= n.ts_start AND greatest(f.ts_start, '2024-01-01') < n.ts_end) OR (greatest(f.ts_start, '2024-01-01') >= n.ts_start AND n.ts_end IS NULL))
	where f.osmid like 'w%' AND jsonb_path_exists(f.tags, '$.voltage ? (@ >= 50000)')
	AND (('2025-11-28' >= f.ts_start AND '2025-11-28' < f.ts_end) OR ('2025-11-28' >= f.ts_start AND f.ts_end is null))
)

SELECT count(distinct nid) FROM list
```

### Evaluate contribution

Get all teams involvement in a given project between two dates

```sql
select team, label, sum(uc.len_delta) as len_delta
from pdm_user_contribs uc
join pdm_projects_teams pt ON pt.userid=uc.userid and pt.project_id=uc.project_id
where uc.project_id=11 and uc.ts BETWEEN '2025-01-01' AND CURRENT_TIMESTAMP
group by team, label
order by team, label;
Count towers involved in >= 50 kV lines at a given date
```
