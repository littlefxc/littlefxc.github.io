---
title: Java8-LocalDateTime记录
date: 2019-08-28 09:38:13
categories: Java
tags:
- LocalDateTime
- Date
---

# Java8-LocalDateTime记录

文中都使用的时区都是东8区，也就是北京时间。这是为了防止服务器设置时区错误时导致时间不对，如果您是其他时区，请自行修改。

## 关键类介绍

– Instant:它代表的是时间戳
– LocalDate:不包含具体时间的日期，比如2019-08-28。它可以用来存储生日，周年纪念日，入职日期等。
– LocalTime:它代表的是不含日期的时间
– LocalDateTime:它包含了日期及时间，不过还是没有偏移信息或者说时区。
– ZonedDateTime:这是一个包含时区的完整的日期时间，偏移量是以UTC/格林威治时间为基准的。

## Java8新增的DateTimeFormatter与SimpleDateFormat的区别

两者最大的区别是，Java8的DateTimeFormatter是线程安全的，而SimpleDateFormat并不是线程安全。

```java
import java.text.DateFormat;
import java.text.SimpleDateFormat;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.Date;

public class Main {

    public static void main(String args[]){

        //解析日期
        String dateStr= "2016年10月25日";
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy年MM月dd日");
        LocalDate date= LocalDate.parse(dateStr, formatter);

        //日期转换为字符串
        LocalDateTime now = LocalDateTime.now();
        DateTimeFormatter format = DateTimeFormatter.ofPattern("yyyy年MM月dd日 hh:mm a");
        String nowStr = now .format(format);
        System.out.println(nowStr);

        //ThreadLocal来限制SimpleDateFormat
        System.out.println(format(new Date()));
    }

    //要在高并发环境下能有比较好的体验，可以使用ThreadLocal来限制SimpleDateFormat只能在线程内共享，这样就避免了多线程导致的线程安全问题。
    private static ThreadLocal<DateFormat> threadLocal = new ThreadLocal<DateFormat>() {
        @Override
        protected DateFormat initialValue() {
            return new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        }
    };

    public static String format(Date date) {
        return threadLocal.get().format(date);
    }

}
```

## LocalDateTime获取毫秒数

```java
//获取秒数
Long second = LocalDateTime.now().toEpochSecond(ZoneOffset.of("+8"));
//获取毫秒数
Long milliSecond = LocalDateTime.now().toInstant(ZoneOffset.of("+8")).toEpochMilli();
```

## LocalDateTime与String互转

```java
//时间转字符串格式化
 DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyyMMddHHmmssSSS");
 String dateTime = LocalDateTime.now(ZoneOffset.of("+8")).format(formatter);

//字符串转时间
String dateTimeStr = "2018-07-28 14:11:15";
DateTimeFormatter df = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
LocalDateTime dateTime = LocalDateTime.parse(dateTimeStr, df);
```

## Date与LocalDateTime互转

```java
//将java.util.Date 转换为java8 的java.time.LocalDateTime,默认时区为东8区
public static LocalDateTime dateConvertToLocalDateTime(Date date) {
    return date.toInstant().atOffset(ZoneOffset.of("+8")).toLocalDateTime();
}

//将java8 的 java.time.LocalDateTime 转换为 java.util.Date，默认时区为东8区
public static Date localDateTimeConvertToDate(LocalDateTime localDateTime) {
    return Date.from(localDateTime.toInstant(ZoneOffset.of("+8")));
}

/**
 * 测试转换是否正确
 */
@Test
public void testDateConvertToLocalDateTime() {
    Date date = DateUtils.parseDate("2018-08-01 21:22:22", DateUtils.DATE_YMDHMS);
    LocalDateTime localDateTime = DateUtils.dateConvertToLocalDateTime(date);
    Long localDateTimeSecond = localDateTime.toEpochSecond(ZoneOffset.of("+8"));
    Long dateSecond = date.toInstant().atOffset(ZoneOffset.of("+8")).toEpochSecond();
    Assert.assertTrue(dateSecond.equals(localDateTimeSecond));
}
```