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
-- update user_behavior_raw set dates=substring(datetimes,1,10),times=substring(datetimes,12,8),hours=substring(datetimes,12,2);
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
第四，为高频查询字段创建索引，通过EXPLAIN的进行性能的识别分析和效果验证，显著提升了基于行为和日期维度的查询效率。约85%的关键查询被这5个索引覆盖，关键查询的执行时间从1分42秒大幅缩短至31秒，性能提升超过70%。
```sql
-- 1. 主查询索引：覆盖行为类型+日期的基础查询
CREATE INDEX idx_behavior_date_user ON userbehavior_raw(behavior_type, dates, user_id, time_stamp);

-- 2. 用户行为分析索引：覆盖用户维度的分析
CREATE INDEX idx_user_date_behavior ON userbehavior_raw(user_id, dates, behavior_type, item_id);

-- 3. 时间序列分析索引：覆盖时间维度的聚合查询
CREATE INDEX idx_date_hour_behavior ON userbehavior_raw(dates, hours, behavior_type);

-- 4. 商品分析索引：覆盖商品和品类的分析
CREATE INDEX idx_item_category_behavior ON userbehavior_raw(item_id, category_id, behavior_type);

-- 5. 留存/用户旅程索引：覆盖用户路径分析
CREATE INDEX idx_user_behavior_sequence ON userbehavior_raw(user_id, behavior_type, dates, time_stamp);
```
第五，数据仓库分层模型，数据仓库分层模型通过将数据处理流程分解为清晰的层次，有效提升了数据管理的系统性和效率。本项目的数据仓库分层模型如下：  
ODS层 (Operational Data Store) - 操作数据层存放原始数据，与源系统保持基本一致，不做或仅做最基础的数据清洗，目的：保留原始数据，便于数据追溯和审计，在本项目中包含userbehavior表。

DWD层 (Data Warehouse Detail) - 数据明细层对原始数据进行清洗、标准化和维度退化，形成规范化的明细数据，为上层分析提供高质量的数据基础，在本项目中包含userbehavior_raw和user_behavior_view，user_behavior_standard两个视图。

DWS层 (Data Warehouse Service) - 数据服务层基于明细数据进行轻度汇总，构建主题宽表以提升查询效率，减少重复计算，在本项目中包含date_hour_behavior和user_behavior_path两个汇总视图。

ADS层 (Application Data Store) - 应用数据层面向具体业务场景进行高度聚合，直接生成各类分析报表和数据看板，包含pv_uv_puv、retention_rate、behavior_user_num等十余张业务报表。

DIM层 (Dimension) - 维度层存放描述性信息和缓慢变化的维度数据，提供统一的维度描述和业务口径，在本项目中包含fanyi这一路径类型翻译表。  
第六，由于单表数据量过亿，因此考虑分区分表分库。例如如下的分区操作：
```sql
-- 按日期分区，每个分区对应一天数据
ALTER TABLE userbehavior_processed 
PARTITION BY RANGE COLUMNS(dates) (
    PARTITION p20171125 VALUES LESS THAN ('2017-11-26'),
    PARTITION p20171126 VALUES LESS THAN ('2017-11-27'),
    PARTITION p20171127 VALUES LESS THAN ('2017-11-28'),
    PARTITION p20171128 VALUES LESS THAN ('2017-11-29'),
    PARTITION p20171129 VALUES LESS THAN ('2017-11-30'),
    PARTITION p20171130 VALUES LESS THAN ('2017-12-01'),
    PARTITION p20171201 VALUES LESS THAN ('2017-12-02'),
    PARTITION p20171202 VALUES LESS THAN ('2017-12-03'),
    PARTITION p20171203 VALUES LESS THAN ('2017-12-04')
);
```
但考虑到数据量虽达到条件，但本身并非巨量且不会再增长，同时经过优化查询时间可以接受，查询模式不适合（混合型查询），数据时间窗口太短（9天），分区带来的写入开销不值得等因素，暂不实施分区分表分库操作，作为后备优化方案以待未来使用。

## 获客情况
```sql
create table userb like userbehavior_raw;
insert into userb
select * from  userbehavior_raw limit 100000;
#pv页面浏览量
#独立访客UV
#浏览深度=PV/UV
select dates,count(*) 'pv',count(distinct user_id) 'uv',round(count(*)/count(distinct user_id),1) 'pv/uv' from userb where behavior_type='pv'
group by dates;
#处理真实数据和保存结果
create table pv_uv_puv(
    dates char(10),
    pv int(9),uv int(9),
    puv decimal(10,1)
);
insert into pv_uv_puv
select dates,count(*) 'pv',count(distinct user_id) 'uv',round(count(*)/count(distinct user_id),1) 'pv/uv' from userbehavior_raw where behavior_type='pv'
group by dates;
delete from pv_uv_puv where dates is null;
```
从数据本身来看，这份统计涵盖了2017年11月25日至12月3日连续九天的用户点击行为概况，主要记录了每日的商品详情页浏览量、独立访客数以及浏览深度三项核心指标。在这期间，页面浏览量呈现先平稳后攀升的态势，初期数值保持在900万上下，最后两日则跃升至1200万以上；独立访客数也表现出类似的增长模式，从最初的约68万逐步上升至93万余。浏览深度指标始终稳定在12.9至13.7的狭窄区间内，显示出每位用户日均浏览页面数的高度一致性。
从分析角度来看，数据揭示了显著的周末效应与潜在促销影响：在12月2日（周六）和3日（周日）两天，无论是页面浏览量还是独立访客数都出现了突兀的峰值增长，这很可能与用户周末闲暇时间增多以及电商平台“双十二”预热活动的开启密切相关。值得注意的是，尽管流量大幅波动，浏览深度却始终保持平稳，这可能意味着用户行为模式具有内在稳定性，平台内容或商品布局对用户吸引力的持续性较好。然而，流量激增并未带动深度指标的同步提升，这也暗示了新涌入用户的参与度可能尚未充分释放，值得进一步观察其后续转化表现。

## 留存和跳失
```sql
#留存率和跳失率
select * from userbehavior_raw where dates is null;
delete from userbehavior_raw where dates is null;

select user_id,dates from userb group by user_id,dates;
#自关联
#自关联全表
select * from
(select user_id,dates from userb group by user_id,dates) a,
(select user_id,dates from userb group by user_id,dates) b
where a.user_id=b.user_id and a.dates<b.dates;
#留存
select a.dates ,
count(if(datediff(b.dates,a.dates)=0,b.user_id,null)) retention_0,
count(if(datediff(b.dates,a.dates)=1,b.user_id,null)) retention_1,
count(if(datediff(b.dates,a.dates)=3,b.user_id,null)) retention_3
from
(select user_id,dates from userb group by user_id,dates) a,
(select user_id,dates from userb group by user_id,dates) b
where a.user_id=b.user_id and a.dates<=b.dates
group by a.dates;

#留存率
select a.dates ,

count(if(datediff(b.dates,a.dates)=1,b.user_id,null))/count(if(datediff(b.dates,a.dates)=0,b.user_id,null)) retention_1

from
(select user_id,dates from userb group by user_id,dates) a,
(select user_id,dates from userb group by user_id,dates) b
where a.user_id=b.user_id and a.dates<=b.dates
group by a.dates;

#保存结果
create table retention_rate(
    dates char(10),
    retention_1 float

);

insert into retention_rate
select a.dates ,

count(if(datediff(b.dates,a.dates)=1,b.user_id,null))/count(if(datediff(b.dates,a.dates)=0,b.user_id,null)) retention_1

from
(select user_id,dates from userbehavior_raw group by user_id,dates) a,
(select user_id,dates from userbehavior_raw group by user_id,dates) b
where a.user_id=b.user_id and a.dates<=b.dates
group by a.dates;
select  * from retention_rate;

#跳失率
select user_id from userbehavior_raw
group by user_id  having  count(behavior_type)=1;

select count(*) from (select user_id from userbehavior_raw
group by user_id  having  count(behavior_type)=1) a;
#流失134767用户
select sum(pv) from pv_uv_puv;
#共4482885
#跳失率134767/4482885 为百分之3.01
```
从数据本身来看，这份留存率报表统计了2017年11月25日至12月3日期间，每日用户的次日留存率与三日（即次次次日）留存率。在数据观测期的前半段（11月25日至11月28日），两项指标表现稳定且优异，次日留存率维持在77%至79%之间，三日留存率也紧随其后，保持在76%至78%附近，用户粘性衰减曲线非常平缓。然而，数据在后半段呈现出明显的模式突变：自11月29日起，三日留存率异常跃升至98%以上；而从12月1日开始，次日留存率同样飙升至98%左右。同时跳失率极低（只浏览一次就消失）为88/89660670。
从分析角度来看，这种突变模式清晰地指向了数据观测窗口的截止效应。由于数据集在12月3日结束，导致对于12月1日及之后来的用户，无法观测到其三日后的行为，因此三日留存率缺失（记为0）；而对于12月1日、2日来的用户，其次日行为恰好在数据期内，故仍有记录。更值得关注的是11月29日、30日那异常高的三日留存率，这很可能是因为其观测日（12月2日、3日）恰逢周末，平台流量与用户活跃度本身就处于峰值，造成了留存率虚高。若排除这些边界干扰，核心结论是：该平台的用户短期留存能力非常出色，用户一旦访问，在接下来几天内持续回访的意愿极强，这构成了平台活跃度的坚实基础。结合此前分析中提到的极低跳失率，共同描绘出一个用户参与深度良好的产品图景。

## 时间序列分析
```sql
#统计日期小时的行为
select dates,hours
,count(if(behavior_type='pv',behavior_type,null)) 'pv'
,count(if(behavior_type='cart',behavior_type,null)) 'cart'
,count(if(behavior_type='fav',behavior_type,null)) 'fav'
,count(if(behavior_type='buy',behavior_type,null)) 'buy'
from userb
group by dates,hours
order by dates,hours;
#存储结果
create table date_hour_behavior(
    dates char(10)
,hours char(2)
,pv int,cart int,fav int,buy int

);
insert into  date_hour_behavior
select dates,hours
,count(if(behavior_type='pv',behavior_type,null)) 'pv'
,count(if(behavior_type='cart',behavior_type,null)) 'cart'
,count(if(behavior_type='fav',behavior_type,null)) 'fav'
,count(if(behavior_type='buy',behavior_type,null)) 'buy'
from userbehavior_raw
group by dates,hours
order by dates,hours
```
从数据本身来看，这份数据集详细记录了2017年11月25日至12月3日连续九天内，每日24小时中用户的四种关键行为数据：页面浏览量、加购次数、收藏次数和购买次数。数据以每小时为粒度进行汇总，共涵盖216个时间点，完整呈现了用户活动在一天内的动态变化。观察原始数值可以发现，各类行为在夜间至凌晨时段普遍处于低谷，而在日间及晚间则显著攀升，且页面浏览量始终占据绝对主导地位，加购、收藏和购买行为的数量级相对较低但趋势相似。
从分析角度来看，数据揭示了清晰的用户行为模式与时间规律：每日的活动高峰通常出现在晚上20点至22点之间，尤其是21点左右达到峰值，这对应于用户晚间休闲时间；而最低谷则集中在凌晨3点至5点，符合常规作息。周末效应尤为突出，12月2日和3日（周六、日）的各项行为指标全面大幅上扬，高峰时段的页面浏览量甚至突破百万，较工作日增长约30%以上，表明用户在有更多自由支配时间时显著提升了电商平台的参与度。此外，加购、收藏与购买行为虽与页面浏览量的走势基本同步，但其转化比例在高峰时段并未显著提升，暗示流量激增时用户的决策可能更倾向于浏览而非立即转化，这为优化促销策略与用户体验提供了重要线索。
## 用户转化率分析
用户转化率分析
```
#用户转化率
select behavior_type,count(distinct user_id) user_num
from userb
group by behavior_type
order by behavior_type desc;
#用户存储结果
create table behavior_user_num(
    behavior_type varchar(5),
    user_num int
);
insert into behavior_user_num
select behavior_type,count(distinct user_id) user_num
from userbehavior_raw
group by behavior_type
order by behavior_type desc;
#行为储存结果
create table behavior_num(
    behavior_type varchar(5),
    behavior_num int
);
insert into behavior_num
select behavior_type,count(*) behavior_num
from userbehavior_raw
group by behavior_type
order by behavior_type desc;
```
从数据本身来看，这份统计从用户数量和行为总数两个维度，汇总了观测期间内四种关键用户行为的整体参与情况。在行为总数上，页面浏览以接近九千万次的绝对优势占据主导，而加购、收藏和购买行为的数量级则依次递减。在用户覆盖面上，超过九十八万用户产生了浏览行为，其中约六十七万用户最终完成了购买，而进行过收藏或加购操作的用户数量介于两者之间。
从分析角度来看，数据揭示了用户从接触到最终转化的典型路径与漏斗效应。几乎所有用户都始于浏览，但只有约四成用户（以收藏用户计）会进入深度互动环节，而最终有超过三分之二的浏览用户完成了购买，这一转化比例显示出平台整体转化效率较为理想。值得注意的是，进行加购的用户数量显著高于收藏用户，这可能意味着加购是更主流、更直接的消费意向表达方式。结合行为总数与用户数的比例来看，每位购买用户平均产生约3次购买行为，而每位收藏用户的平均收藏次数约为7.4次，这反映了用户在决策过程中的重复比价或心仪商品积累的习惯。整体而言，用户行为漏斗从浏览到购买的留存率较高，但如何提升用户从浏览向深度互动（收藏/加购）环节的转化，或许是进一步优化运营的关键。
## 行为路径分析
```sql
create view user_behavior_view as
select user_id,item_id
,count(if(behavior_type='pv',behavior_type,null)) pv
,count(if(behavior_type='fac',behavior_type,null)) fac
,count(if(behavior_type='cart',behavior_type,null)) cart
,count(if(behavior_type='buy',behavior_type,null)) buy
from userbehavior_raw
group by user_id, item_id;
create view user_behavior_standard as
select user_id,item_id
,(case when pv>0 then 1 else 0 end ) 浏览了
,(case when fac>0 then 1 else 0 end ) 收藏了
,(case when cart>0 then 1 else 0 end ) 加购了
,(case when buy>0 then 1 else 0 end ) 购买了
from user_behavior_view;
#路径类型
create view user_behavior_path as
select *,
concat(浏览了, 收藏了, 加购了,购买了) 购买路径类型
from user_behavior_standard as a
where a.购买了>0;
#统计路径的数量
create view path_count as
select 购买路径类型,count(*) 数量
from user_behavior_path
group by 购买路径类型
order by 数量 desc;
#类型代码翻译
create table fanyi(
    path_type char(4),
    description varchar(40)
);
insert into fanyi
values ('0001','直接购买'),('1001','浏览后购买'),('0011','加购后购买'),
       ('0101','收藏后购买'),('1101','浏览收藏后购买'),('1011','浏览加购后购买')
,('0111','加购收藏后购买'),('1111','浏览加购收藏后购买');

select * from fanyi a right join path_count b on a.path_type=b.购买路径类型;

#存储
CREATE TABLE path_result(
    path_tape char(4),
    description varchar(40),
    num int
);
insert into path_result
select a.path_type, description,数量 from fanyi a right join path_count b
on a.path_type=b.购买路径类型;
select * from path_count;
select  sum(buy) from user_behavior_view
where cart=0 and fac=0 and buy >0
#直接浏览购买量为99880
```
这份数据从用户与商品交互的微观视角，系统归纳并量化了八种不同的购买决策路径。数据显示，最主流的购买路径是“浏览后直接购买”，共发生了近95万次，紧随其后的是没有任何前期互动行为的“直接购买”，超过了51万次。相比之下，那些包含了加购、收藏等深度互动环节的决策路径，虽然组合多样，但其发生频率均显著降低，其中同时经历浏览、加购、收藏全过程的“完整体验”路径仅出现了约1.3万次。
从分析角度来看，用户的购买行为呈现出高度集中且决策直接的特征。超过80%的购买行为通过“浏览后购买”和“直接购买”这两种最简路径完成，这强烈暗示了在本次观测场景下，用户要么目的明确、快速下单，要么在简单浏览后便迅速做出购买决策，整体购物流程倾向于高效和直接。虽然存在加购、收藏等中间环节，但它们并非主流购买路径的必要前置步骤，更多是作为少数用户的辅助决策工具。这一方面可能说明平台的商品展示或推荐机制有效地促成了快速转化，另一方面也提示我们，那些复杂的、包含多重考虑环节的购物行为模式在当前用户群体中并不普遍，未来或可通过引导用户使用加购和收藏功能，来进一步沉淀消费意向并提升复购率。

## RFM模型
```
#RFM模型 R最近一次消费 F消费频率 M值消费金额
select user_id,max(dates) 最近购买时间 ,count(behavior_type) 购买次数
from userbehavior_raw
where behavior_type='buy'
group by user_id
order by  3 desc;
create table rfm_model(
    user_id int,
    recent char(10),
    frequency int
);
insert into rfm_model
select user_id,max(dates) 最近购买时间 ,count(behavior_type) 购买次数
from userbehavior_raw
where behavior_type='buy'
group by user_id
order by  3 desc;

alter table rfm_model add column   fscore int;
update  rfm_model set fscore= case
    when frequency between 9 and 10 then 5
when frequency between 7 and 8 then 4
when frequency between 5 and 6 then 3
when frequency between 3 and 4 then 2
else 1 end ;

alter table rfm_model add column   rscore int;
update  rfm_model set rscore= case
    when recent ='2017-12-03' then 5
 when recent in ('2017-12-01','2017-12-02') then 4
when recent in ('2017-11-29','2017-11-30')  then 3
when recent in ('2017-11-27','2017-11-28')  then 2
else 1 end ;

set @f_avg=null;
set @r_avg=null;
select avg(rscore) into @r_avg from rfm_model;
select avg(fscore) into @f_avg from rfm_model;

alter table rfm_model add class varchar(40);
update rfm_model set class= case
    when rfm_model.fscore>@f_avg and rfm_model.rscore>@r_avg
    then '价值用户'
when rfm_model.fscore>@f_avg and rfm_model.rscore<@r_avg
    then '保持用户'
when rfm_model.fscore<@f_avg and rfm_model.rscore>@r_avg
    then '发展用户'
when rfm_model.fscore<@f_avg and rfm_model.rscore<@r_avg
    then '挽留用户'
end;
select class ,count(user_id) from rfm_model
group by class;
```
基于RFM模型的SQL定义，用户被划分为四类：价值用户指消费频率高且最近消费时间近的用户，属于核心优质客户；保持用户指消费频率高但最近消费时间较远的用户，需关注其活跃度以防流失；发展用户指最近消费时间近但消费频率较低的用户，具有成长潜力；挽留用户指消费频率低且最近消费时间远的用户，存在流失风险，需主动干预。
这份基于RFM模型的用户分类结果呈现出清晰的用户结构分布。其中，发展用户数量最多，达到288,203人，挽留用户紧随其后，有261,969人，而价值用户为97,724人，保持用户则最少，仅24,508人。这一分布表明，该平台的用户中具备高消费频率且近期活跃的“价值用户”占比并不占主导，相反，大量用户属于近期有互动但消费频次不高的“发展用户”，以及近期未消费且消费频次偏低的“挽留用户”，反映出用户整体活跃性与消费深度仍有较大提升空间，用户结构呈现一定的长尾特征。
从分析角度来看，该结果提示平台用户中存在显著的留存与转化机会。发展用户规模较大，意味着许多用户虽近期有过消费，但尚未形成稳定的购买习惯，可通过激励策略进一步培养其消费频次，使其向价值用户迁移。而挽留用户数量庞大，则说明存在相当一部分沉默或流失风险较高的用户，需要主动采取召回与互动措施，防止其完全流失。整体而言，该分类为后续精细化运营提供了明确的方向——在维护好现有价值用户的同时，应着力提升发展用户的消费黏性，并有效唤醒挽留用户，以优化整体用户健康度与平台价值。

## 商品分析
```
商品按热度分类
select category_id,count(if(behavior_type='pv',behavior_type,null)) '品类浏览量'
from userbehavior_raw
group by  category_id
order by 2 desc
limit 10;

select item_id,count(if(behavior_type='pv',behavior_type,null)) '商品浏览量'
from userbehavior_raw
group by  item_id
order by 2 desc
limit 10;
#各个品类的最热商品
select category_id,item_id,品类商品浏览量
    from
(select category_id,item_id,count(if(behavior_type='pv',behavior_type,null)) '品类商品浏览量',
rank() over (partition by category_id order by count(if(behavior_type='pv',behavior_type,null))) r
from userbehavior_raw
group by  category_id,item_id
order by 3 desc) a
where r=1
order by 品类商品浏览量
limit 10
;
-- 存储

create table popular_categories(
category_id int,
pv int);
create table popular_items(
item_id int,
pv int);
create table popular_cateitems(
category_id int,
item_id int,
pv int);


insert into popular_categories
select category_id
,count(if(behavior_type='pv',behavior_type,null)) '品类浏览量'
from userbehavior_raw
GROUP BY category_id
order by 2 desc
limit 10;


insert into popular_items
select item_id
,count(if(behavior_type='pv',behavior_type,null)) '商品浏览量'
from userbehavior_raw
GROUP BY item_id
order by 2 desc
limit 10;

insert into popular_cateitems
select category_id,item_id,
品类商品浏览量 from
(
select category_id,item_id
,count(if(behavior_type='pv',behavior_type,null)) '品类商品浏览量'
,rank()over(partition by category_id order by count(if(behavior_type='pv',behavior_type,null)) desc) r
from userbehavior_raw
GROUP BY category_id,item_id
order by 3 desc
) a
where a.r = 1
order by a.品类商品浏览量 desc
limit 10;


select * from popular_categories;
select * from popular_items;
select * from popular_cateitems;

-- 特定商品转化率
#购买用户占总用户数量
-- 保存
create table item_detail(
item_id int,
pv int,
fav int,
cart int,
buy int,
user_buy_rate float);

insert into item_detail
select item_id
,count(if(behavior_type='pv',behavior_type,null)) 'pv'
,count(if(behavior_type='fav',behavior_type,null)) 'fav'
,count(if(behavior_type='cart',behavior_type,null)) 'cart'
,count(if(behavior_type='buy',behavior_type,null)) 'buy'
,count(distinct if(behavior_type='buy',user_id,null))/count(distinct user_id) 商品转化率
from userbehavior_raw
group by item_id
order by 商品转化率 desc;

select * from item_detail;

-- 品类转化率

create table category_detail(
category_id int,
pv int,
fav int,
cart int,
buy int,
user_buy_rate float);

insert into category_detail
select category_id
,count(if(behavior_type='pv',behavior_type,null)) 'pv'
,count(if(behavior_type='fav',behavior_type,null)) 'fav'
,count(if(behavior_type='cart',behavior_type,null)) 'cart'
,count(if(behavior_type='buy',behavior_type,null)) 'buy'
,count(distinct if(behavior_type='buy',user_id,null))/count(distinct user_id) 品类转化率
from userbehavior_raw
group by category_id
order by 品类转化率 desc;

select * from category_detail;
```
我们提取了浏览量最高的前10名商品和品类，并以CSV格式存储了相关数据。具体而言，商品数据如popular_items.csv所示，其中商品ID 812879以30,079次浏览位居首位，而其他商品如3845720、138964等浏览量也均超过14,000次；品类数据则如popular_categories.csv所示，品类ID 4756105以4,477,682次浏览显著领先，其余前10名品类的浏览量从超过315万次到约73万次不等，这些数据直观反映了用户行为在商品和品类层面的集中分布。
从分析角度看，商品和品类的浏览量呈现出明显的长尾效应，热门项目的浏览量远高于其他。例如，顶级商品的浏览量达到30,079次，而前10名商品的浏览量均超过14,000次，显示用户兴趣高度集中于少数热门商品；品类层面同样如此，最高浏览量的品类接近450万次，与前10名中最低的约73万次形成鲜明对比，这揭示了用户行为在电商平台中的聚焦趋势。这些高浏览量数据可作为隐式反馈的重要指标，用于优化推荐系统的算法，通过识别用户偏好提升个性化服务；同时，数据覆盖的9天时间窗口可能暗示了季节性促销或短期用户行为模式，为进一步研究用户动态和商业策略提供了基础。


## 随机森林得出的规则分析
规则名称 触发条件 预警精准度 用户覆盖率 业务解读与运营建议 
沉睡用户预警 days_since_last_active > 1 且 total_actions < 20 53.9% 51.8% （最高优先级） 捕捉“低活跃且已开始沉默”的广泛群体。建议进行自动化、低成本的批量触达，如推送个性化内容或小额优惠券。 
高放弃率-低活跃预警 cart_abandon_rate > 0.6 且 active_days <= 2 52.0% 26.7% 针对低活跃用户中有明确“购买意向受挫”信号的子集。可进行稍高成本的定向干预，如购物车商品提醒专享券。 
低转化-不稳定预警 pv_to_cart_rate < 0.1 且 std_active_hour > 3 42.5% 47.3% 精准度较低，建议作为参考观察或与首要规则组合使用。
基于随机森林模型的用户流失预警规则分析结果如下：
从模型提取的三条核心预警规则来看，规则的有效性和覆盖范围呈现明显差异：
沉睡用户预警规则表现最优：该规则以53.9%的预警精准度和51.8%的用户覆盖率成为最高优先级。触发条件"days_since_last_active > 1且total_actions < 20"准确捕捉了"低活跃且已开始沉默"的用户群体。其高覆盖率表明超过一半的用户存在此风险特征，建议采用自动化、低成本的批量触达策略，如推送个性化内容或小额优惠券，以最低成本挽回最大用户基数。
高放弃率-低活跃预警规则精准度较高但覆盖有限：52.0%的预警精准度显示模型能有效识别"购买意向受挫"的用户，但仅26.7%的覆盖率意味着这部分用户属于特定子集。触发条件"cart_abandon_rate > 0.6且active_days <= 2"针对的是有明确购物意图但最终放弃的低活跃用户。建议采用稍高成本的定向干预策略，如购物车商品提醒专享券，针对性强但需控制成本。
低转化-不稳定预警规则需审慎应用：该规则虽然覆盖了47.3%的广泛用户群体，但42.5%的预警精准度相对较低。触发条件"pv_to_cart_rate < 0.1且std_active_hour > 3"反映用户行为模式不稳定的特征。建议将其作为辅助观察指标或与其他规则组合使用，避免单独应用导致误判。
综合分析结论：这三条规则构建了从广泛到精准的用户流失预警体系，沉睡用户预警应作为核心干预重点，高放弃率预警可作为补充精准策略，低转化预警则提供辅助参考，形成分层分级的精细化用户留存运营方案。

## 可视化
形成可视化图表15副
## 建议
一、针对用户活跃时间规律，优化营销与推送策略
高峰时段强化曝光：在每日20:00–22:00（尤其是21:00）及周末全天，增加首页推荐位更新频率，推出限时优惠、直播带货等互动活动，提升用户参与深度。
低谷时段试探唤醒：在凌晨3:00–5:00等低活跃时段，可通过推送个性化、低干扰的内容（如“明日上新预告”“收藏商品降价提醒”），试探唤醒潜在用户。
二、提升“浏览→深度互动”转化，优化用户决策路径
加强收藏/加购引导：在商品详情页增设明显引导按钮，并结合激励策略（如“收藏后降价提醒”“加购享专属优惠券”），提升用户从浏览到收藏/加购的转化率。
简化购买路径：鉴于用户倾向于直接购买或浏览后立即购买，应持续优化“一键下单”“快速复购”等功能，减少中间步骤，提升购买效率。
三、基于RFM分层，实施精细化用户运营
发展用户→价值用户：针对近期有消费但频次低的发展用户（占比最大），可通过积分奖励、频次任务（如“月内二次购物赠券”）提升其消费黏性。
挽留用户→激活召回：对长期未消费的挽留用户，通过个性化推送（如“您可能错过的商品”“老用户专享回归礼”）+ 低门槛优惠，尝试唤醒其购物意愿。
价值用户→持续维护：为价值用户提供VIP服务、新品优先体验、高价值专属客服等，增强其忠诚度与复购意愿。
四、依据预警规则，建立分层干预机制
高覆盖沉睡用户：针对“低活跃且开始沉默”的用户（覆盖率51.8%），实施低成本自动化触达，如每日签到奖励、个性化内容推荐，保持用户连接。
高放弃用户定向激励：对加购后放弃购买的用户（尤其低活跃者），及时推送“购物车商品专享券”或库存提醒，降低放弃率。
低转化用户行为观察：对行为不稳定、转化率低的用户，暂不作为独立干预对象，可将其行为数据纳入长期观察与模型迭代。
五、强化品类与商品维度推荐，提升内容吸引力
热门品类与商品资源倾斜：对浏览量最高的品类（如4756105）和商品（如812879），在相关推荐、搜索排名中给予加权，并围绕其开发专题活动或捆绑销售。
长尾品类潜力挖掘：通过“相似推荐”“小众好物”等模块，引导用户探索非热门品类，平衡流量分布，提升整体商品曝光效率。
六、持续监测周末与促销期表现，动态调整策略
周末流量高峰前置准备：在周五提前预热周末活动，确保系统承压能力，并准备弹性营销内容以承接流量。
促销期间转化跟踪：在类似“双十二”等大促期间，重点监控浏览→购买转化率变化，及时调整优惠力度与界面引导，避免流量浪费。
## AB测试方案
基于报告中路径分析的发现——超过80%的购买行为通过“浏览后直接购买”或“直接购买”完成，而包含加购的决策路径占比相对较低——这揭示了用户倾向于快速、直接的决策模式，但同时也暗示了“加购”这一深度互动环节的潜力未被充分释放，用户加购后可能因决策中断或动力不足而放弃购买。
为验证这一洞察并量化干预措施的效果，我设计了一套A/B测试方案，旨在科学评估“对加购后放弃的用户推送专属优惠券”这一策略的有效性。该方案的核心假设是：对于已产生加购行为但未在24小时内下单的用户，一条及时的、具吸引力的专享券提醒，能够有效弥补其决策动力缺口，从而显著提升从加购到购买的转化率。
实验将随机选取在观测周期内发生加购行为但24小时内未购买的用户，并将其均匀分为实验组与对照组。实验组用户将在加购满24小时后的首个活跃时段（如当晚20点）收到一张针对其加购商品的限时专享优惠券推送；对照组用户则遵循现有流程，不接收任何额外激励。实验的主要评估指标是“加购后7天内的购买转化率”，同时将“平均客单价”和“优惠券核销后的订单毛利率”作为关键的护栏指标，以确保提升转化率的同时不损害整体商业价值。
为确保结果的统计显著性，我们基于历史数据中加购用户的自然转化率（可依据报告中的行为数据估算），结合期望检测的最小提升效应（例如相对提升15%），计算出所需的用户样本量，并设定为期两周的实验周期以覆盖完整的用户决策窗口。在数据分析阶段，我们将采用双比例Z检验来比较两组的转化率差异，并以p值小于0.05作为统计显著性的标准。
此外，方案包含了严谨的风险控制措施。例如，会先进行小流量的灰度测试以验证推送系统的稳定性；设定明确的提前终止规则，若实验初期数据显示客单价出现大幅下滑或用户反馈显著负面，则立即停止实验。通过这套完整的A/B测试方案，不仅能为“高放弃用户定向激励”的建议提供数据驱动的决策依据，更展现了将分析洞察转化为可验证、可落地的科学实验的闭环能力。


