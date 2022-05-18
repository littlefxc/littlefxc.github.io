---
title: Mysql中日期比较大小的方法
date: 2019-10-08 18:18:15
categories: RDBMS
tags:
- mysql
---

假如有个表`commodity`有个字段add_time,它的数据类型为datetime,有人可能会这样写sql：

```sql
select * from product where add_time = '2013-01-12';
```

这种语句，如果你存储的格式是YY-mm-dd这样，那么OK，
如果你存储的格式是：2018-01-12 23:23:56 这种就悲剧了，此时你可以用 DATE() 函数用来返回日期的部分；sql如下处理：

  ```sql
  select * from product where Date(add_time) = '2018-01-12'
  ```

1. 如果你要查询2017年1月份加入的产品呢？

  ```sql
  select * from product where date(add_time) between '2013-01-01' and '2013-01-31'
  ```

  或者： 还可以这样写：

  ```sql
  select * from product where Year(add_time) = 2013 and Month(add_time) = 1
  ```

2. 其date_col的值是在最后30天以内：

  ```sql
  SELECT * FROM commodity  WHERE TO_DAYS( NOW() ) - TO_DAYS( date_col )  <= 30;
  ```

3. DAYOFWEEK(date) ： 返回日期date的星期索引(1=星期天, 7=星期六) 这些索引值对应于ODBC标准。

  ```sql
  select DAYOFWEEK('1998-02-03');                     ->  3
  ```

4. WEEKDAY(date)  ： 返回date的星期索引 (0=星期一, 6= 星期天)

  ```sql
   select WEEKDAY('1997-10-04 22:23:00');             -> 5
   select WEEKDAY('2018-07-07');                      -> 5
  ```

5. DAYOFMONTH(date) ：  返回date的月份中日期，在1到31范围内。

  ```sql
  select DAYOFMONTH('1998-02-03');                    -> 3
  ```

6. DAYOFYEAR(date) ：  返回date在一年中的日数, 在1到366范围内。

  ```sql
  select DAYOFYEAR('1998-02-03');                     -> 34
  ```

7. MONTH(date) ：   返回date的月份，范围1到12。

  ```sql
  select MONTH('1998-02-03');                         -> 2
  ```

8. DAYNAME(date) ： 返回date的星期名字。

  ```sql
  select DAYNAME("1998-02-05");                       -> 'Thursday'
  select DAYNAME("20180707");                         -> 'Saturday'
  ```

1. MONTHNAME(date) ： 返回date的月份名字。

  ```sql
  select MONTHNAME("1998-02-05");                     -> 'February'
  ```

10. QUARTER(date)  ： 返回date一年中的季度，范围1到4。

  ```sql
  select QUARTER('98-04-01');                          -> 2
  select QUARTER('20180707');                          -> 3
  ```

这是做统计数据时候用了的sql.

查询今天、昨天、一周内、8周、12周等数据 直接在sql写时间查询。 避免时区转化。数据库我们使用的UTC时间。

- Today

  ```sql
  SELECT str_to_date(curdate(),'%Y-%m-%d %H:%i:%s') as todayBegin;
  SELECT date_add(date_add(str_to_date(curdate(),'%Y-%m-%d %H:%i:%s'),interval 1 DAY),INTERVAL -1 SECOND) as todayEnd;
  ```

- 昨天

  ``sql
  SELECT str_to_date(date_sub(curdate(), interval 1 day),'%Y-%m-%d %H:%i:%s') as yestodayBegin;
  SELECT date_add(date_add(str_to_date(date_sub(curdate(), interval 1 day),'%Y-%m-%d %H:%i:%s'),interval 1 DAY),INTERVAL -1 SECOND) as yestodayEnd;
  ```

- 最近7天 Last Seven days

  ```sql
  SELECT str_to_date(date_sub(curdate(), interval 6 day),'%Y-%m-%d %H:%i:%s') as lastSevenDaysBegin;
  SELECT date_add(date_add(str_to_date(curdate(),'%Y-%m-%d %H:%i:%s'),interval 1 DAY),INTERVAL -1 SECOND) as todayEnd;
  ```
      
- 最近14天 Last Fourteen days

  ```sql
  SELECT str_to_date(date_sub(curdate(), interval 13 day),'%Y-%m-%d %H:%i:%s') as lastFourteenDaysBegin;
  SELECT date_add(date_add(str_to_date(curdate(),'%Y-%m-%d %H:%i:%s'),interval 1 DAY),INTERVAL -1 SECOND) as todayEnd;
  ```

- 最近30天 Last Thirty days

  ```sql
  SELECT str_to_date(date_sub(curdate(), interval 29 day),'%Y-%m-%d %H:%i:%s') as lastThirtyDaysBegin;
  SELECT date_add(date_add(str_to_date(curdate(),'%Y-%m-%d %H:%i:%s'),interval 1 DAY),INTERVAL -1 SECOND) as todayEnd;
  ```

- 最近8周 Last eight weeks

  ```sql
  SELECT str_to_date(date_sub(curdate(), interval 8 week),'%Y-%m-%d %H:%i:%s') as lastEightWeeksBegin;
  SELECT date_add(date_add(str_to_date(curdate(),'%Y-%m-%d %H:%i:%s'),interval 1 DAY),INTERVAL -1 SECOND) as todayEnd;
  ```

- 12 最近12周 Last twelve weeks

  ```sql
  SELECT str_to_date(date_sub(curdate(), interval 12 week),'%Y-%m-%d %H:%i:%s') as lastTwelveWeeksBegin;
  SELECT date_add(date_add(str_to_date(curdate(),'%Y-%m-%d %H:%i:%s'),interval 1 DAY),INTERVAL -1 SECOND) as todayEnd;
  ```

- 最近3月 Last three month

  ``sql
  SELECT str_to_date(date_sub(curdate(), interval 3 month),'%Y-%m-%d %H:%i:%s') as lastThreeMonthBegin;
  ```

- 未来3天 Three day later

  ```sql
  SELECT str_to_date(date_sub(curdate(), interval -3 day),'%Y-%m-%d %H:%i:%s') as threeDaysLaterEnd;
  ```

- 未来3月 Three months later

  ```sql
  SELECT str_to_date(date_sub(curdate(), interval -3 month),'%Y-%m-%d %H:%i:%s') as threeMonthsLaterEnd; 
  ```

- 取一天的开始时间

  ```sql
  SELECT str_to_date(DATE_FORMAT('2018-03-03','%Y-%m-%d'),'%Y-%m-%d %H:%i:%s');
  ```

- 取第二天的开始时间

  ```sql
  select DATE_ADD(str_to_date(DATE_FORMAT(NOW(),'%Y-%m-%d'),'%Y-%m-%d %H:%i:%s'),INTERVAL 1 DAY);
  select DATE_ADD(str_to_date(DATE_FORMAT('2018-03-03','%Y-%m-%d'),'%Y-%m-%d %H:%i:%s'),INTERVAL 1 DAY);
  ```

- 取一天的结束时间

  ```sql
  select DATE_ADD(DATE_ADD(str_to_date(DATE_FORMAT(NOW(),'%Y-%m-%d'),'%Y-%m-%d %H:%i:%s'),INTERVAL 1 DAY),INTERVAL -1 SECOND);
  select DATE_ADD(DATE_ADD(str_to_date(DATE_FORMAT('2018-03-03','%Y-%m-%d'),'%Y-%m-%d %H:%i:%s'),INTERVAL 1 DAY),INTERVAL -1 SECOND);
  ```
