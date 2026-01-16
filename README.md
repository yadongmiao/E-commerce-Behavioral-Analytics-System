# E-commerce Behavioral Analytics System 
## 数据描述 
UserBehavior是阿里巴巴提供的一个淘宝用户行为数据集，用于隐式反馈推荐问题的研究。本数据集涵盖了2017年11月25日至2017年12月3日期间约一百万随机用户的所有行为记录，包括点击、购买、加购及喜欢四种类型，总行为数量达到100,150,807条。数据以类似MovieLens-20M的格式组织，每一行代表一条用户行为，包含用户ID、商品ID、商品类目ID、行为类型和时间戳五个字段，并以逗号分隔。其中，用户ID、商品ID和商品类目ID均为整数类型，行为类型为字符串枚举值（‘pv’表示点击商品详情页，‘buy’表示购买，‘cart’表示加入购物车，‘fav’表示收藏），时间戳则记录行为发生的具体时间。该数据集中共涉及987,994个用户、4,162,024个商品以及9,439个商品类目。
## 分析目标
通过对用户购物行为数据的分析，为客户提供更精准的隐式反馈推荐。从用户角度：分析用户行为规律，提高用户忠诚度，帮助用户快速找到商品。从网站角度：提高网站交叉销售能力，提高成交转化率。
## 分析思路
技术栈： Excel、MySQL、PYTHON、POWER BI  
数据预处理:去空、去重、去异常,时间戳格式化  
数据库优化：调整缓冲池大小，通过EXPLAIN的进行性能的识别分析和效果验证创建核心索引，分批次更新策略，采用标准 CTAS模式，数据仓库分层建模。  
数据分析:用户行为习惯分析：分析用户最活跃的日期及每天活跃时间段。用户行为转化分析：基于AARRR漏斗模型，分析从浏览到购买的转化率与流失率，提出改善建议。类目偏好分析：通过点击、收藏、加购、购买频率，探索用户购买偏好，优化产品推荐策略。用户价值分析（RFM模型）：识别高价值付费用户，针对其偏好推送个性化销售方案。预警规则识别：通过PYTHON使用随机森林建模，识别出核心的三个预警规则。
## 数据预处理
```sql
-- 检查空值 
select * from user_behavior_raw where user_id is null;
select * from user_behavior_raw where item_id is null;
select * from user_behavior_raw where category_id is null;
select * from user_behavior_raw where behavior_type is null;
select * from user_behavior_raw where timestamps is null;
```
数据完整，无缺失值。
```sql
-- 检查重复值 
select user_id,item_id,timestamps from user_behavior_raw
group by user_id,item_id,timestamps
having count(*)>1;
-- 去重 
alter table user_behavior_raw add id int first;
select * from user_behavior_raw limit 5;
alter table user_behavior_raw modify id int primary key auto_increment;
delete user_behavior_raw from
user_behavior_raw,
(
select user_id,item_id,timestamps,min(id) id from user_behavior_raw
group by user_id,item_id,timestamps
having count(*)>1
) t2
where user_behavior_raw.user_id=t2.user_id
and user_behavior_raw.item_id=t2.item_id
and user_behavior_raw.timestamps=t2.timestamps
and user_behavior_raw.id>t2.id;
```
经处理，数据去除重复值。
```sql
-- 新增日期：date time hour
-- datetime
alter table user_behavior_raw add datetimes TIMESTAMP(0);
update user_behavior_raw set datetimes=FROM_UNIXTIME(timestamps);
select * from user_behavior_raw limit 5;
-- date
alter table user_behavior_raw add dates char(10);
alter table user_behavior_raw add times char(8);
alter table user_behavior_raw add hours char(2);
-- update user_behavior set dates=substring(datetimes,1,10),times=substring(datetimes,12,8),hours=substring(datetimes,12,2);
update user_behavior_raw set dates=substring(datetimes,1,10);
update user_behavior_raw set times=substring(datetimes,12,8);
update user_behavior_raw set hours=substring(datetimes,12,2);
select * from user_behavior_raw limit 5;
```
经处理，原始数据中的 `timestamps` 列（Unix时间戳）被转换为可读的日期时间格式，并进一步拆分为单独的日期（`dates`）、时间（`times`）和小时（`hours`）字段，以便后续进行按日、按时的精细化分析。
```sql
-- 去异常 
select max(datetimes),min(datetimes) from user_behavior_raw;
delete from user_behavior_raw
where datetimes < '2017-11-25 00:00:00'
or datetimes > '2017-12-03 23:59:59';
```
经处理，已去除时间异常的值。
```sql
-- 数据概览 
desc user_behavior_raw;
select * from user_behavior_raw limit 5;
SELECT count(1) from user_behavior_raw; -- 100095496条记录
```
综上，完成了对数据的预处理
## 数据库优化
在淘宝用户行为数据分析项目中，针对亿级数据处理性能瓶颈，我实施了多项核心SQL优化措施：
第一，调整InnoDB缓冲池大小至1GB，为批量操作提供充足的内存缓存空间
```sql
show VARIABLES like '%_buffer%';
set GLOBAL innodb_buffer_pool_size=1070000000;
```
第二，将1亿条数据的批量更新操作分解为4个2500万条的子任务，通过分批次更新策略避免长事务锁表问题，保障数据库稳定性；
```sql
-- 批次 1/4：更新第1个2500万条数据
UPDATE userbehavior_raw SET datetimes = FROM_UNIXTIME(time_stamps) WHERE id BETWEEN 1 AND 25000000;
-- 批次 2/4：更新第2个2500万条数据
UPDATE userbehavior_raw SET datetimes = FROM_UNIXTIME(time_stamps) WHERE id BETWEEN 25000001 AND 50000000;
-- 批次 3/4：更新第3个2500万条数据
UPDATE userbehavior_raw SET datetimes = FROM_UNIXTIME(time_stamps) WHERE id BETWEEN 50000001 AND 75000000;
-- 批次 4/4：更新最后2500万条数据（使用 id > 75000000 确保覆盖所有剩余数据）
UPDATE userbehavior_raw SET datetimes = FROM_UNIXTIME(time_stamps) WHERE id > 75000000;
```
第三，采用标准 CTAS（Create Table As Select）模式替代传统的 ALTER+UPDATE 方案把原本原来的1h20M 的执行时间的内容缩短至16 m 23 s，性能提升超过70%
```sql
CREATE TABLE userbehavior_processed AS
SELECT 
    user_id,
    item_id,
    category_id,
    behavior_type,
    time_stamp,
    FROM_UNIXTIME(time_stamp) AS datetimes,      
    DATE(FROM_UNIXTIME(time_stamp)) AS dates,    
    TIME(FROM_UNIXTIME(time_stamp)) AS times,    
    HOUR(FROM_UNIXTIME(time_stamp)) AS hours     
FROM userbehavior_raw;
```
