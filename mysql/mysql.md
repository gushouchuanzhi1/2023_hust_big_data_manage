a) . JSON 基本查询: 

1.在business表中，查询city位于Tampa的商户所有信息，按被评论数降序排序，限制返回10条。

```mysql
SELECT * 

FROM business 

WHERE JSON_UNQUOTE(JSON_EXTRACT(business_info, '$.city')) = 'Tampa' 

ORDER BY JSON_EXTRACT(business_info, '$.review_count') DESC 

LIMIT 10;
```

​                               

2. 在business表中,查询前五条记录的business_info列和

business_info中attributes的所有键，以Json数组形式返回，同时返回对应键的数量。



```
SELECT 
JSON_KEYS(JSON_EXTRACT(business_info, '$.attributes')) AS attribute_keys, JSON_LENGTH(JSON_EXTRACT(business_info, '$.attributes')) AS attribute_count
FROM business

LIMIT 5;
```

 

3. 在business表中,查询该表business_info列中name,stars, attributes的内容和JSON类型,限制返回行数为5。 

```
SELECT

  JSON_UNQUOTE(JSON_EXTRACT(business_info, '$.name')) AS name,

  JSON_TYPE(JSON_EXTRACT(business_info, '$.name')) AS name_type,

  JSON_UNQUOTE(JSON_EXTRACT(business_info, '$.stars')) AS stars,

  JSON_TYPE(JSON_EXTRACT(business_info, '$.stars')) AS stars_type,

  JSON_UNQUOTE(JSON_EXTRACT(business_info, '$.attributes')) AS attributes,

  JSON_TYPE(JSON_EXTRACT(business_info, '$.attributes')) AS attributes_type

FROM business

LIMIT 5;
```

 

4.在business表中,查询拥有电视("HasTV"是"True")且星期天不营业

(Sunday键不存在,当然也可以hours键不存在,即is null)的商户,

返回它的名字(不带双引号),属性,营业时间,并按名字升序排序,限制10条记录.正确

```
SELECT JSON_UNQUOTE(JSON_EXTRACT(business_info, '$.name')) AS name, JSON_UNQUOTE(JSON_EXTRACT(business_info, '$.attributes')) AS attributes, JSON_UNQUOTE(JSON_EXTRACT(business_info, '$.hours')) AS hours

FROM business

WHERE JSON_UNQUOTE(JSON_EXTRACT(business_info, '$.attributes.HasTV')) = 'True' AND (JSON_UNQUOTE(JSON_EXTRACT(business_info, '$.hours.Sunday')) IS NULL OR JSON_UNQUOTE(JSON_EXTRACT(business_info, '$.hours')) IS NULL)

ORDER BY name ASC

LIMIT 10;
```

 

5.使用explain查看select * from user where user_info->'$.name'='Wanda'的

执行计划,其中执行计划按JSON格式输出;并且实际执行一次该查询,请注意观察语句

消耗的时间并与MongoDB的查询方式进行对比(MongoDB要执行此查询要求,相应

的语句是什么?执行计划是怎样的?并给出查询效率对比).



 

b）json的增删修改

6. 在busines表中,查询id为4r3Ck65DCG1T6gpWodPyrg的商户business_info,这 里对info列的显示需要使用JSON_PRETTY(business_info)让可读性更高

查询原来状态：

```
SELECT JSON_PRETTY(JSON_UNQUOTE(JSON_EXTRACT(business_info, '$'))) AS business_info

FROM business

WHERE business_id = '4r3Ck65DCG1T6gpWodPyrg';
```

原来状态：

 

修改，对比：

-- 查询修改前的数据

```
SELECT JSON_PRETTY(JSON_UNQUOTE(JSON_EXTRACT(business_info, '$'))) AS business_info_before

FROM business

WHERE business_id = '4r3Ck65DCG1T6gpWodPyrg';
```

 

-- 更新商户数据

```
UPDATE business

SET business_info = JSON_SET(

  business_info,

  '$.hours.Tuesday', '16:0-23:0',

  '$.stars', 4.5,

  '$.attributes.WiFi', 'Free'

)

WHERE business_id = '4r3Ck65DCG1T6gpWodPyrg';
```

 

-- 查询修改后的数据

```
SELECT JSON_PRETTY(JSON_UNQUOTE(JSON_EXTRACT(business_info, '$'))) AS business_info_after

FROM business

WHERE business_id = '4r3Ck65DCG1T6gpWodPyrg'
```

;

 

7. 向 business 表插入一个 id 是'aaaaaabbbbbbcccccc2023'的商户,其商户信息与 id 为'5d-fkQteaqO6CSCqS5q4rw'的商户完全一样,插入完成之后,将这个新记录 的 info 中的 name 键值对删去,最后查询'aaaaaabbbbbbcccccc2023'的所有信息.

 

```
INSERT INTO business (business_id, business_info)

SELECT 'aaaaaabbbbbbcccccc2023', business_info

FROM business

WHERE business_id = '5d-fkQteaqO6CSCqS5q4rw';
```

 

 

 

 

删除name键值对：

```
UPDATE business

SET business_info = JSON_REMOVE(business_info, '$.name')

WHERE business_id = 'aaaaaabbbbbbcccccc2023';
```

查询

```
SELECT *

FROM business

WHERE business_id = 'aaaaaabbbbbbcccccc2023';
```

 

 



 

C）json聚合

8. 在 business 表的所有商户中,按所在州(state)进行聚合,对于每个州返还一个 JSON 对象,这个对象的每一个键值对中,key 是城市,value 是城市总共出现的 次数,结果按照州名升序排序. 提示:这里需要去掉引号让 group by 的 key 更好一些

```
SELECT

  state,

  JSON_OBJECTAGG(city, city_count) AS city_occurrences

FROM (

  SELECT

​    TRIM(BOTH '"' FROM JSON_UNQUOTE(JSON_EXTRACT(business_info, '$.state'))) AS state,

​    TRIM(BOTH '"' FROM JSON_UNQUOTE(JSON_EXTRACT(business_info, '$.city'))) AS city,

​    COUNT(*) AS city_count

  FROM business

  GROUP BY state, city

) AS city_counts

GROUP BY state

ORDER BY state;
```

 

9.查询user_id是'__1cb6cwl3uAbMTK3xaGbg'的用户的所有朋友的建议,并按用户 进行分组聚合,对每一个用户,返还用户的id,用户的名字,由他/她的所有建议构成的 字符串数组,最后按名字升序排序输出。

```
Select u.user_id as user_id,
 json_unquote(json_extract(u.user_info, ‘$.name’)) as name,
 json_arrayagg(t.tip_info->‘$.text’) as text_array
 from ‘user’ u
 left join tip t on t.user_id = u.user_id
 where regexp_like(u.user_info->‘$.friends‘, ’__1cb6cwl3uAbMTK3xaGbg‘)
 group by user_id, name
 order by name asc; 
```

 

 

d) . JSON实用函数的使用: 

10.在business表中,分别查询城市在EdMonton和Elsmere的商铺,并使用 JSON_OVERLAPS()判断在这两个城市的商铺之间是否在一周中至少有一天营业时间完全重合(即hours对象中至少有一个键值对相同,不考虑键不存在的情况),是返回 1,不是返回0。

```
SELECT

  b1.business_info->'$.name' AS name_1,

  b1.business_info->'$.city' AS city_1,

  b1.business_info->'$.hours' AS hours_1,

  b2.business_info->'$.name' AS name_2,

  b2.business_info->'$.city' AS city_2,

  b2.business_info->'$.hours' AS hours_2,

  IF(

​    (

​      JSON_UNQUOTE(JSON_EXTRACT(b1.business_info->'$.hours', '$.Monday')) = JSON_UNQUOTE(JSON_EXTRACT(b2.business_info->'$.hours', '$.Monday'))

​      OR (JSON_UNQUOTE(JSON_EXTRACT(b1.business_info->'$.hours', '$.Monday')) = "0:0-0:0" AND JSON_UNQUOTE(JSON_EXTRACT(b2.business_info->'$.hours', '$.Monday')) = "0:0-0:0")

​    )

​    AND JSON_UNQUOTE(JSON_EXTRACT(b1.business_info->'$.hours', '$.Tuesday')) = JSON_UNQUOTE(JSON_EXTRACT(b2.business_info->'$.hours', '$.Tuesday'))

​    AND JSON_UNQUOTE(JSON_EXTRACT(b1.business_info->'$.hours', '$.Wednesday')) = JSON_UNQUOTE(JSON_EXTRACT(b2.business_info->'$.hours', '$.Wednesday'))

​    AND JSON_UNQUOTE(JSON_EXTRACT(b1.business_info->'$.hours', '$.Thursday')) = JSON_UNQUOTE(JSON_EXTRACT(b2.business_info->'$.hours', '$.Thursday'))

​    AND JSON_UNQUOTE(JSON_EXTRACT(b1.business_info->'$.hours', '$.Friday')) = JSON_UNQUOTE(JSON_EXTRACT(b2.business_info->'$.hours', '$.Friday'))

​    AND JSON_UNQUOTE(JSON_EXTRACT(b1.business_info->'$.hours', '$.Saturday')) = JSON_UNQUOTE(JSON_EXTRACT(b2.business_info->'$.hours', '$.Saturday'))

​    AND JSON_UNQUOTE(JSON_EXTRACT(b1.business_info->'$.hours', '$.Sunday')) = JSON_UNQUOTE(JSON_EXTRACT(b2.business_info->'$.hours', '$.Sunday'))

  , 1, 0) AS has_same_opentime

FROM business AS b1

JOIN business AS b2

ON b1.business_id != b2.business_id

WHERE b1.business_info->'$.city' = 'EdMonton'

AND b2.business_info->'$.city' = 'Elsmere';
```

 

11. 在user表中,查询funny大于2000且平均评分大于4.0的用户,返回他们的名字,平均评分,以及按json数组形式表示的funny, useful, cool,三者的和,限制10条;尝试按平均评分降序排序,并使用explain查看排序的开销,与第一题的排序情况做对比, 主要关注"rows_examined_per_scan"和"cost_info"

每次会话调整设置：SET sort_buffer_size = 10000000;

```
SELECT 

  JSON_UNQUOTE(JSON_EXTRACT(user_info, '$.name')) AS name,

  JSON_UNQUOTE(JSON_EXTRACT(user_info, '$.average_stars')) AS avg_stars,

  JSON_ARRAY(

​    JSON_UNQUOTE(JSON_EXTRACT(user_info, '$.funny')),

​    JSON_ARRAY(

​    CAST(JSON_UNQUOTE(JSON_EXTRACT(user_info, '$.funny')) AS SIGNED) + CAST(JSON_UNQUOTE(JSON_EXTRACT(user_info, '$.useful')) AS SIGNED) + CAST(JSON_UNQUOTE(JSON_EXTRACT(user_info, '$.cool')) AS SIGNED)

  ) AS '[funny, useful, cool, sum]'

FROM 

user      

JSON_UNQUOTE(JSON_EXTRACT(user_info, '$.funny')),

  JSON_UNQUOTE(JSON_EXTRACT(user_info, '$.useful')),

  JSON_UNQUOTE(JSON_EXTRACT(user_info, '$.cool')),

  CAST(JSON_UNQUOTE(JSON_EXTRACT(user_info, '$.funny')) AS SIGNED) + CAST(JSON_UNQUOTE(JSON_EXTRACT(user_info, '$.useful')) AS SIGNED) + CAST(JSON_UNQUOTE(JSON_EXTRACT(user_info, '$.cool')) AS SIGNED)

  ) AS '[funny, useful, cool, sum]'

FROM 

   user

WHERE 

   CAST(JSON_UNQUOTE(JSON_EXTRACT(user_info, '$.funny')) AS SIGNED) > 2000

   AND CAST(JSON_UNQUOTE(JSON_EXTRACT(user_info, '$.average_stars')) AS DECIMAL(3, 2)) > 4.0

LIMIT 10;
```

 

 

12. 在 tip 表中找到被提建议最多的商户和提出建议最多的用户,合并二者的 info 列的 JSON 文档为一个文档显示,对于 JSON 文档中相同的 key 值,应该保留二 者的 value 值

 

```
SELECT
  json_pretty(json_object(
   ’business_with_most_tips’,
   json_object(‘business_id’, t1.business_id, ’tips_count’, t1.max_tips_count),
   ’business_info’, 
   t1.business_info,
   ’user_with_most_tips’,
   json_object(‘user_id’, t2.user_id, ’tips_count’, t2.max_tips_count),
   ’user_info’, 
   t2.user_info SELECT

 json_pretty(json_object(

  'business_with_most_tips',

  json_object('business_id', t1.business_id, 'tips_count', t1.max_tips_count),

  'business_info', 

  t1.business_info,

  'user_with_most_tips',

  json_object('user_id', t2.user_id, 'tips_count', t2.max_tips_count),

  'user_info', 

  t2.user_info

 )) AS merged_info

FROM (

 SELECT t.business_id, COUNT(*) AS max_tips_count, b.business_info

 FROM tip t

 JOIN business b ON t.business_id = b.business_id

 GROUP BY t.business_id

 ORDER BY max_tips_count DESC

 LIMIT 1

) AS t1

JOIN (

 SELECT t.user_id, COUNT(*) AS max_tips_count, u.user_info

 FROM tip t

 JOIN user u ON t.user_id = u.user_id

 GROUP BY t.user_id

 ORDER BY max_tips_count DESC

 LIMIT 1

) AS t2; 
```

 

 

13. 查询被评论数前三的商户,使用 JSON_TABLE()导出他们的名字,被评论数,是 否在星期二营业(即"hours"中有"Tuesday"的键值对,是返回 1,不是返回 0),和一 周所有的营业时段(不考虑顺序,一个时段就对应一行,对每个商户,从 1 开始对 这些时段递增编号), 最后按商户名字升序排序.

```
WITH BusinessInfo AS (

 SELECT

  JSON_UNQUOTE(JSON_EXTRACT(business_info, '$.name')) AS business_name,

  CAST(JSON_UNQUOTE(JSON_EXTRACT(business_info, '$.review_count')) AS SIGNED) AS business_review_count,

  IF(JSON_UNQUOTE(JSON_EXTRACT(business_info->'$.hours', '$.Tuesday')), 1, 0) AS business_open_on_Tuesday,

  JSON_EXTRACT(business_info, '$.hours') AS business_hours

 FROM business

 ORDER BY business_review_count DESC

 LIMIT 3

) 

SELECT

 business_name,

 business_review_count,

 business_open_on_Tuesday,

 num,

 hours_in_a_week

FROM (

 SELECT

  business_name,

  business_review_count,

  business_open_on_Tuesday,

  num,

  hours_in_a_week

 FROM BusinessInfo

 CROSS JOIN JSON_TABLE(

  business_hours,

  '$.*' COLUMNS (

   num FOR ORDINALITY,

   hours_in_a_week VARCHAR(100) PATH '$'

  )

 ) AS hours

) AS SubQuery

ORDER BY business_name ASC; 
```

