```SQL
/**
update sales profiles
for Freemium
*/

/**
-- generate unchanged sales records
with tt as
(
select sp.data->>'accountId' as "accountId",
sp.data->>'locationId' as "locationId",
sp.data->>'id' as "salesProfileId",
map->>'marketCode' as "legacyMarketCode",
map->>'marketCode' as "newMarketCode",
map->>'categoryCode' as "categroyCode",
'UNCHANGED' as "marketStatus"
from storefronts sf inner join sales_profiles sp
on sf.data->>'id' = sp.data->>'storefrontId',
jsonb_array_elements(sp.data->'services') as map
where
sf.data->>'accountId' <> '00000000-0000-0000-0000-000000000000'
and sf.data->>'statusCode' = 'LIVE'
and sf.data->>'isPurchased' = 'false'
order by sp.data->>'id'
limit 100)

insert into free_storefront_market_map2
select row_to_json(tt) from tt;

-- generate update sales records
with tt as
(
select sp.data->>'accountId' as "accountId",
sp.data->>'locationId' as "locationId",
sp.data->>'id' as "salesProfileId",
map->>'marketCode' as "legacyMarketCode",
map->>'marketCode' || '-new' as "newMarketCode",
map->>'categoryCode' as "categroyCode",
'UPDATE' as "marketStatus"
from storefronts sf inner join sales_profiles sp
on sf.data->>'id' = sp.data->>'storefrontId',
jsonb_array_elements(sp.data->'services') as map
where
sf.data->>'accountId' <> '00000000-0000-0000-0000-000000000000'
and sf.data->>'statusCode' = 'LIVE'
and sf.data->>'isPurchased' = 'false'
limit 100)

insert into free_storefront_market_map2
select row_to_json(tt) from tt;
*/


-- insert count number to free_storefront_market_map2
-- this to determite which service should be remove
with salesCount as(
select data->>'salesProfileId' as "salesProfileId",count(1)
from free_storefront_market_map2
where data->>'marketStatus'='UPDATE'
group by data->>'salesProfileId'
)

update free_storefront_market_map2 fs
set data=jsonb_set(
fs.data,
'{count}',
s.count::text::jsonb
)
from salesCount s
where fs.data->>'salesProfileId'=s."salesProfileId";

-- temp sales records
drop table if exists sales_temp;

with salesids as (
	select distinct data->>'salesProfileId' as "salesProfileId"
	from free_storefront_market_map2
)

select sp.data->>'id' as id, map as data into sales_temp from sales_profiles sp
inner join salesids on salesids."salesProfileId" = sp.data->>'id',
jsonb_array_elements(sp.data->'services') as map


/**
select * From free_storefront_market_map2;

select data->>'updated', data->>'statusCode', * from sales_temp
order by id;

-- select * into sales_temp_0523 from sales_temp;
-- drop table sales_temp
-- select * into sales_temp from sales_temp_0523;
-- update the statusCode and marketCode
*/

-- update for the remvoed service
update sales_temp st
set data=
jsonb_set(
	jsonb_set(
	st.data,
	'{statusCode}',
	'"LIVE"'
	),
	'{updated}',
	'true'
)
from free_storefront_market_map2 fs
where fs.data->>'salesProfileId' = st.id
and fs.data->>'legacyMarketCode' = st.data->>'marketCode'
and fs.data->>'marketStatus' = 'UPDATE'
and st.data->>'statusCode' = 'REMOVED'
and coalesce(st.data->>'updated', '') <> 'true'

-- update the statusCode and marketCode
update sales_temp st
set data=
jsonb_set(
	jsonb_set(
		jsonb_set(
			st.data,
			'{statusCode}',
			('"' || CASE WHEN fs.data->>'count' <> '1' THEN 'REMOVED' ELSE st.data->>'statusCode' END || '"')::jsonb
		),
		'{updated}',
		'true'
	),
	'{marketCode}',
	('"' || CASE WHEN fs.data->>'count' <> '1' THEN fs.data->>'legacyMarketCode' ELSE fs.data->>'newMarketCode' END || '"')::jsonb
)
from free_storefront_market_map2 fs
where fs.data->>'salesProfileId' = st.id
and fs.data->>'legacyMarketCode' = st.data->>'marketCode'
and fs.data->>'marketStatus' = 'UPDATE'
and st.data->>'statusCode' <> 'REMOVED'
and coalesce(st.data->>'updated', '') <> 'true'

-- update sales profiles
update sales_profiles_0523 sp1
set data=jsonb_set(
    sp.data,
    '{services}',
    newval
)
 from sales_profiles_0523 sp
 inner join sales_temp st on sp.data->>'id'=st.id,
 LATERAL (
     select jsonb_agg(data) as newval
     from sales_temp
     where id=sp.data->>'id'
 ) services
 where sp.data->>'id'=sp1.data->>'id';

```
