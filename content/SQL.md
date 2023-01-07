# 常用MySQL语法
1. 列举所有的测风塔  
```
SELECT DISTINCT site FROM zb_wind_data;

> +------+
  | site |
  +------+
  | 0305 |
  | 0307 |
  +------+
```
2. 查询0305测风塔所有的70m风速风向数据，按时间从早到晚排序（其他高度，其他测风塔同理）  
```
SELECT date,time,70m_v_avg,70m_deg_avg
FROM zb_wind_data  
WHERE site="0305"
ORDER BY date ASC,time ASC;

> |  2016-1-1  | 00:10:00 |       7.2 |         171 |
       ...         ...         ...          ...
  | 2016-12-31 | 23:20:00 |       6.5 |         236 |
  | 2016-12-31 | 23:30:00 |         8 |         239 |
  | 2016-12-31 | 23:40:00 |         7 |         227 |
  | 2016-12-31 | 23:50:00 |       8.3 |         235 |
+------------+----------+-----------+-------------+
52147 rows in set (0.23 sec)
```
3. 查询0305测风塔的70m风速数据总条数  
```
SELECT COUNT(70m_v_avg)
AS total_num
FROM zb_wind_data
WHERE site="0305";

> +-----------+
  | total_num |
  +-----------+
  |     52147 |
  +-----------+
```
4. 查询0305测风塔的70m风速在[3,5)区间内的计数  
```
SELECT COUNT(70m_v_avg)
AS selected_num
FROM zb_wind_data
WHERE site="0305"
AND 70m_v_avg>=3
AND 70m_v_avg<5;

> +--------------+
  | selected_num |
  +--------------+
  |        12509 |
  +--------------+
```
5. 查询最新时间前一小时的0305测风塔的70m风速风向数据
```
SELECT str_to_date(CONCAT(date,' ',time),'%Y-%m-%d %H:%i:%s') AS datetime,
70m_v_avg AS speed,
70m_deg_avg AS degree
FROM zb_wind_data
WHERE site="0305" 
AND TIMESTAMPDIFF(
  HOUR,
  str_to_date(CONCAT(date,' ',time),'%Y-%m-%d %H:%i:%s'),
  (
    SELECT
    MAX(str_to_date(CONCAT(date,' ',time),'%Y-%m-%d %H:%i:%s'))
    FROM zb_wind_data
  )
)<1
ORDER BY date ASC,time ASC;

> +---------------------+-----------+-------------+
  | 2016-12-31 23:00:00 |       6.2 |         231 |
  | 2016-12-31 23:10:00 |       6.3 |         234 |
  | 2016-12-31 23:20:00 |       6.5 |         236 |
  | 2016-12-31 23:30:00 |         8 |         239 |
  | 2016-12-31 23:40:00 |         7 |         227 |
  | 2016-12-31 23:50:00 |       8.3 |         235 |
  +---------------------+-----------+-------------+
```
6. 查询指定日期时间区间内(2016.10.18 23:10-2016.10.19 00:40)的0305测风塔的70m风速风向数据  
```
SELECT str_to_date(CONCAT(date,' ',time),'%Y-%m-%d %H:%i:%s') AS datetime,
70m_v_avg AS speed,
70m_deg_avg AS degree
FROM zb_wind_data
WHERE site="0305"
AND str_to_date(CONCAT(date,' ',time),'%Y-%m-%d %H:%i:%s')
BETWEEN str_to_date('2016-9-28 23:0:0','%Y-%m-%d %H:%i:%s')
AND str_to_date('2016-10-2 01:0:0','%Y-%m-%d %H:%i:%s')
ORDER BY date ASC,time ASC;

> +---------------------+-------+--------+
  | datetime            | speed | degree |
  +---------------------+-------+--------+
  | 2016-09-28 23:00:00 |   5.8 |    178 |
  | 2016-09-28 23:10:00 |   5.6 |    175 |
  | 2016-09-28 23:20:00 |   5.9 |    172 |
  | 2016-09-28 23:30:00 |   6.6 |    173 |
  | 2016-09-28 23:40:00 |   7.5 |    174 |
  | 2016-09-28 23:50:00 |   7.7 |    173 |
  | 2016-09-29 00:00:00 |   7.8 |    173 |
  | 2016-09-29 00:10:00 |     8 |    173 |
  |         ...         |  ...  |   ...  |
```
7. 在所有数据中查询0305测风塔70m的风向频率统计
```
SELECT ROUND(70m_deg_avg/22.5)%16 AS direction,
COUNT(70m_v_avg) AS f
FROM zb_wind_data
WHERE site="0305"
GROUP BY ROUND(70m_deg_avg/22.5)%16;

> +-----------+------+
  | direction | f    |
  +-----------+------+
  |         0 | 4161 |
  |         1 | 3549 |
  |         2 | 1440 |
  |         3 |  295 |
  |         4 |  227 |
  |         5 |  436 |
  |         6 |  903 |
  |         7 | 3608 |
  |         8 | 8693 |
  |         9 | 6376 |
  |        10 | 2225 |
  |        11 | 1854 |
  |        12 | 2432 |
  |        13 | 4484 |
  |        14 | 5415 |
  |        15 | 6049 |
  +-----------+------+
```

8. 查询日期范围内0305测风塔的70m风速风向数据，对在[3-5)m/s的风速，按风向所在扇区分组统计频率
``` 
SELECT ROUND(70m_deg_avg/22.5)%16 AS direction,
COUNT(70m_v_avg)
/
(
  SELECT COUNT(70m_v_avg)
  FROM zb_wind_data
  WHERE site="0305"
  AND 70m_v_avg>=3
  AND 70m_v_avg<15
  AND str_to_date(CONCAT(date,' ',time),'%Y-%m-%d %H:%i:%s') >  '2016-10-18 23:0:0'
  AND str_to_date(CONCAT(date,' ',time),'%Y-%m-%d %H:%i:%s') <  '2016-11-19 00:0:0'
) AS f
FROM zb_wind_data
WHERE site="0305"
AND 70m_v_avg>=3
AND 70m_v_avg<15
AND str_to_date(CONCAT(date,' ',time),'%Y-%m-%d %H:%i:%s') >=  '2016-10-18 23:0:0'
AND str_to_date(CONCAT(date,' ',time),'%Y-%m-%d %H:%i:%s') <  '2016-11-19 00:0:0'
GROUP BY ROUND(70m_deg_avg/22.5)%16;

> +-----------+--------+
  | direction | f      |
  +-----------+--------+
  |         0 | 0.0496 |
  |         1 | 0.0255 |
  |         2 | 0.0082 |
  |         3 | 0.0007 |
  |         4 | 0.0015 |
  |         5 | 0.0052 |
  |         6 | 0.0119 |
  |         7 | 0.0655 |
  |         8 | 0.1860 |
  |         9 | 0.1309 |
  |        10 | 0.0687 |
  |        11 | 0.0687 |
  |        12 | 0.0806 |
  |        13 | 0.0754 |
  |        14 | 0.0965 |
  |        15 | 0.1252 |
  +-----------+--------+
```
9. 查询所有的高度字段（要求既有风速又有风向）
```
SELECT height
FROM (
  SELECT SUBSTRING_INDEX(
    COLUMN_NAME,'_',1
  ) AS height
  FROM information_schema.COLUMNS
  WHERE TABLE_NAME = 'zb_wind_data'
  AND (
    COLUMN_NAME LIKE '%_v_avg'
    OR
    COLUMN_NAME LIKE '%_deg_avg'
  )
) AS T
GROUP BY height
HAVING COUNT(height)=2;

> +--------+
  | height |
  +--------+
  | 10m    |
  | 40m    |
  | 70m    |
  +--------+

```
10. 查询0305测风塔70m高度第1扇区（NNE）在2016-2-1 13:00至2016-2-10 19:00时间范围内的风速分布
```
SELECT FLOOR(70m_v_avg) AS rangeStart,
COUNT(FLOOR(70m_v_avg)) AS count
FROM zb_wind_data
WHERE site="0305"
AND ROUND(70m_deg_avg/22.5)%16=1
AND str_to_date(CONCAT(date,' ',time),'%Y-%m-%d %H:%i:%s') >= '2016-2-1 13:0:0'
AND str_to_date(CONCAT(date,' ',time),'%Y-%m-%d %H:%i:%s') < '2016-2-10 19:0:0'
GROUP BY FLOOR(70m_v_avg);

> +------------+-------+
  | rangeStart | count |
  +------------+-------+
  |          2 |     2 |
  |          4 |     1 |
  |          5 |     5 |
  |          6 |     1 |
  |          7 |     1 |
  +------------+-------+
```
11. 按照最近某段时间不同的时间粒度（1小时/1天）返回速度均值
```
SELECT str_to_date(
  CONCAT(date,' ',HOUR(time),':',MINUTE(time),':',SECOND(time)),'%Y-%m-%d %H:%i:%s'
  ) AS datetime,
ROUND(AVG(70m_v_avg),2) AS speed,
ROUND(AVG(70m_v_max),2) AS maxSpeed,
ROUND(AVG(70m_v_min),2) AS minSpeed,
ROUND(AVG(70m_deg_avg),2) AS degree
FROM zb_wind_data
WHERE site="0305" 
AND TIMESTAMPDIFF(
  HOUR,
  str_to_date(CONCAT(date,' ',time),'%Y-%m-%d %H:%i:%s'),
  (
    SELECT
    MAX(str_to_date(CONCAT(date,' ',time),'%Y-%m-%d %H:%i:%s'))
    FROM zb_wind_data
  )
)<1
GROUP BY datetime
ORDER BY datetime ASC;

>
+---------------------+-------------------+--------+
| datetime            | speed             | degree |
+---------------------+-------------------+--------+
| 2016-12-31 23:00:00 | 6.199999809265137 |    231 |
| 2016-12-31 23:10:00 | 6.300000190734863 |    234 |
| 2016-12-31 23:20:00 |               6.5 |    236 |
| 2016-12-31 23:30:00 |                 8 |    239 |
| 2016-12-31 23:40:00 |                 7 |    227 |
| 2016-12-31 23:50:00 | 8.300000190734863 |    235 |
+---------------------+-------------------+--------+
```