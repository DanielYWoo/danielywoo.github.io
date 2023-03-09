---
layout: post
title:  "Develop Applications with Time - Floating and Explicit Time"
date:   2021-01-26 10:39:00
published: true
---

# Overview

When developing applications for users around the world, time is often implemented incorrectly, that's because most developers don't understand the concepts of Floating Time and Explicit Time. In this article, we will discuss what they are and how to use them with time zones.


# Floating Time and Explicit Time

A "floating date/time" is a time value that isn't tied to a specific time zone. e.g, Birthday. Your birthday is Jan 1st in China, if you move to the US, your birthday is still Jan 1st but the actual UTC epoch time is different. It's better to have a date string like “2020-1-1” to represent it. Another example is Christmas, '2021-12-25T00:00' happens differently in different time zones.

An “explicit date/time” or “non-floating time” has zone or offset info, it represents an actual moment, and only happens once no matter which time zone you are in. To achieve that, you either have all your explicit dates in UTC epoch, or have explicit time zone information in your date types. e.g, an ISO8601 date string like "2021-01-26T01:00:00Z+08". In Java, there are two similar classes for time zones: ZoneId and ZoneOffset. ZoneId is named like "Asia/Beijing" and it has DST. ZoneOffset is much simpler, it's just a number-like offset, e.g, '+08'. Correspondingly there are two DateTimes in Java: OffsetDateTime and ZonedDateTime. We will talk about them in detail later.

Obviously, if you mix them up, storing a rocket launch time in a floating date time field, you probably will launch the rocket 8 hours ahead of schedule. That's why you see astronauts always say Houston time in movies, they want explicit time. And you probably will never say your Birthday is in Berlin time since it's a floating time, if you fly to a different country in a different time zone you won't adjust your birthday several hours back.

There could be conversions between floating time and explicit time. e.g, to send a congrats email on your birthday which is a floating time, your application can detect which time zone the user is in, then calculate the explicit time to send it. Another example is to send a meeting request to people in different time zones, then it must be an explicit time, but you need to convert that into each participant's local time (floating) at the front end. In this article, we mainly focus on how to handle explicit time correctly because this is much more difficult than handling floating time.

Explicit time can be processed and displayed correctly only if 

1) you store it correctly in database or cache, 

2) you have the correct conversion between the storage and your application, 

3) you have the correct conversion between your application and clients like a browser or an Android device.

In this section, we will discuss the three items you need to pay attention to.


# Date Time Storage in Database

Let's demonstrate this problem with the most popular database - MySQL.

* DATETIME: Eight bytes: A four-byte integer for date packed as YYYY×10000 + MM×100 + DD and a four-byte integer for time packed as HH×10000 + MM×100 + SS
* TIMESTAMP: A four-byte integer representing seconds UTC since the epoch ('1970-01-01 00:00:00' UTC)

In MySQL, DATETIME is a floating date type, Timestamp is supposed to be an explicit date type, it will convert time into UTC when saving, and convert back to your local time when loading. It sounds good but it's wrongly implemented, when you retrieve a row from the database there is no time zone in the returned time string. I will explain it in more detail with some tests.

Now let's do some tests to play around with the two date types.

First, we create a table with datetime and timestamp columns.

```sql
CREATE TABLE `test_date` (
  `C1` bigint DEFAULT NULL,
  `C2` datetime DEFAULT NULL,
  `C3` timestamp NULL DEFAULT NULL
) ENGINE=InnoDB;

```

You can set the default time zone to +10 in mysqld, then by default all connections/sessions will use this default time zone.
```mysql
[mysqld]
default-time-zone="+10:00"
```
This can be overriden per connection/session by
```sql
SET @@time_zone = '+10:00'
```

Now, let's switch to the time zone UTC+2, insert a row then switch to UTC+3.
```sql
set @@time_zone ='+2:00';
insert into test_date values(1, '2021-01-26T01:00', '2021-01-26T01:00');
select * from test_date;

Output:
1	2021-01-26 01:00:00	2021-01-26 01:00:00

set @@time_zone ='+3:00';
select * from test_date;

Output:
1	2021-01-26 01:00:00	2021-01-26 02:00:00
```
You can see DATETIME does not have the concept of time zone so no matter what time zone the client sets, it returns the same string "2021-01-26 01:00:00". Timestamp is aware of the time zone when inserting, it converts the time string into UTC epoch and stores it in the database. Later when you retrieve it back in a new time zone, it converts the UTC epoch into the new timezone as a new string plus one hour.

So, it looks like a good idea to use TIMESTAMP since it's explicit for rocket launching, right? Actually no, because TIMESTAMP does not have time zone information when you retrieve it back from the database, MySQL should be designed to return explicit time with a time zone like this:
```
2021-01-26 01:00:00+2
```
or just return a UTC epoch although it's harder to read by humans. JDBC driver can get the UTC epoch time by getTimestamp(index), but unfortunately, this is also implemented wrongly in JDBC, and we will discuss it in the next section.

If you export data from a MySQL instance and import it to another instance, no matter what time zone the server is in, DATETIME fields don't change, but TIMESTAMP fields change. If you have all date fields as DATETIME and save UTC time into them, your application will be working just fine.

# Conversion between Application and Database

As of now, after JDK8, OffsetDateTime and ZonedDateTime are recommended date types in your Java application for explicit date types, and you can use them for the stock exchange, rocket launch schedule etc; LocalDateTime is a new class for floating time, it does not have time zone information internally, you can use it for hotel morning call, birthday congrats email, etc.

Example to use ZonedDateTime

```java
ZonedDateTime zonedDateTime1 = ZonedDateTime.now(ZoneId.of("Europe/Berlin"));
ZonedDateTime zonedDateTime2 = ZonedDateTime.now(ZoneId.of("Asia/Shanghai"));
System.out.println(zonedDateTime1 + "\tEpoch=" + zonedDateTime1.toEpochSecond());
System.out.println(zonedDateTime2 + "\tEpoch=" + zonedDateTime2.toEpochSecond());

Output:
2021-01-25T07:26:26.225154+01:00[Europe/Berlin]	Epoch=1611555986
2021-01-25T14:26:26.228058+08:00[Asia/Shanghai]	Epoch=1611555986
```
You can see that the two ZonedDateTime instances have exactly the same UTC Epoch, but different hour field.

Example to use OffsetDateTime
```java
OffsetDateTime offsetDateTime1 = OffsetDateTime.now(ZoneOffset.of("+1"));
OffsetDateTime offsetDateTime2 = OffsetDateTime.now(ZoneOffset.of("+8"));
System.out.println(offsetDateTime1 + "\tEpoch=" + offsetDateTime1.toEpochSecond());
System.out.println(offsetDateTime2 + "\tEpoch=" + offsetDateTime2.toEpochSecond());

Output:
2021-01-25T07:29:46.228025+01:00	Epoch=1611556186
2021-01-25T14:29:46.228541+08:00	Epoch=1611556186
```
You can see that the two OffsetDateTime instances have exactly the same behavior as ZonedDateTime. The only difference to ZoneDateTime is that the former uses named ZoneId (and probably with DST) and the latter uses ZoneOffset to represent time zones.

Unfortunately, OffsetDateTime/ZonedDateTime are not supported by MySQL JDBC driver because they are explicit date types, while DATETIME is a floating date type, and there is no time zone information in it. For an application with users around the world in different time zones, you have to save/load UTC time with DATETIME to simulate explicit time. You probably need the old good Date to interact with MySQL since it's an explicit UTC epoch actually. For a floating time, use LocalDateTime.  The table below shows that you can only use the first two date types in Java with MySQL.

| Java Date Type | Explicit or Floating   | Save | Load |
| -------------- | ---------------------- | ---- | ---- |
| Date/Timestamp | Explicit as Epoch      | Y    | Y    |
| LocalDateTIme  | Floating               | Y    | Y    |
| OffsetDateTime | Explicit with TZ/Epoch | N    | N    |
| ZonedDateTime  | Explicit with TZ/Epoch | N    | N    |


Here are some code examples for explicit time via JDBC:
```java
pst.setTimestamp(2, new Timestamp(System.currentTimeMillis())); // correct UTC epoch
pst.setTimestamp(3, new Timestamp(utcEpochFromFrontEnd));       // correct UTC epoch
pst.setTimestamp(2, new Timestamp(parse("2021-01-01T01:00+2")));// should be able to convert to correct UTC epoch with "+2"
```
The examples above work for an explicit time because the Timestamp instances are correct UTC Epoch timestamps. But the code below won't work, because the function parse(...) has no idea if this string is in the current time zone or UTC, most likely it cannot convert it to the correct UTC Epoch time. Just remember, the front end application should always submit a UTC epoch timestamp in long type, or an ISO8601 date string with time zone explicitly.
```java
pst.setTimestamp(2, new Timestamp(parse("2021-01-01T01:00"))); // this won't work for explicit time
```

If you want to handle floating time, try LocalDateTime on DATETIME field like this:
```java
pst.setObject(2, LocalDateTime.now());
pst.setString(2, "2021-01-26T01:00:00"); // a string without time zone information
```

# Let's test

Now let's write a simple Java application to do two things: first, save some data into the table we just created, then set the session to a different time zone and read it back.

```java
conn.createStatement().execute("truncate table test_date");
conn.createStatement().execute("set @@time_zone = '+00:00';");

PreparedStatement pst = conn.prepareStatement("insert into test_date (c1, c2, c3) values(?,?,?)");

pst.setInt(1, 1);
pst.setTimestamp(2, new Timestamp(System.currentTimeMillis()));
pst.setTimestamp(3, new Timestamp(System.currentTimeMillis()));
pst.executeUpdate();

conn.createStatement().execute("set @@time_zone = '+01:00';");
rs = conn.createStatement().executeQuery("select * from test_date");
rs.next();

public static void print(String type, int i, ResultSet rs) throws SQLException {
    System.out.print(type);
    System.out.print("\tUTC Epoch: " + rs.getTimestamp(i).getTime());
    System.out.print("\tDate: " + rs.getDate(i));
    System.out.println("\tLocalDateTime: " + rs.getObject(i, LocalDateTime.class));
}

-- Output for time zone +0:00
DATETIME	UTC Epoch: 1611655784000	Date: 2021-01-25	LocalDateTime: 2021-01-26T20:09:44
TIMESTAMP	UTC Epoch: 1611655784000	Date: 2021-01-25	LocalDateTime: 2021-01-26T20:09:44

-- Output for time zone +1:00
DATETIME	UTC Epoch: 1611655784000	Date: 2021-01-25	LocalDateTime: 2021-01-26T20:09:44
TIMESTAMP	UTC Epoch: 1611659384000	Date: 2021-01-25	LocalDateTime: 2021-01-26T21:09:44
```

From the output you can see DATETIME works as expected, saving and loading with UTC, it can be used as explicit time. But TIMESTAMP does not.

First of all, the LocalDateTime returned by TIMESTAMP has one hour adjusted, it is expected behavior of TIMESTAMP in MySQL, but this is useless, it's a wrong design of MySQL. It is not a floating time, and this not an explicit time either. For floating date, the hour should not be adjusted. For explicit time, we should have time zone information in LocalDateTime but this class does not have a zone.

Moreover, the epoch returned by getTimestamp(i) is <b>wrong</b>. Since Epoch is UTC, no matter what time zone the client is in, we should always see the same UTC epoch, but in our test, you see the epoch time is changed from 1611655784000 to 1611659384000.

Why is that? Why the epoch is wrong for TIMESTAMP? Let's see the class MysqlTextValueDecoder in JDBC driver.

```java
int year = getInt(bytes, offset, offset + 4);
int month = getInt(bytes, offset + 5, offset + 7);
int day = getInt(bytes, offset + 8, offset + 10);
int hours = getInt(bytes, offset + 11, offset + 13);
int minutes = getInt(bytes, offset + 14, offset + 16);
int seconds = getInt(bytes, offset + 17, offset + 19);
...
return new InternalTimestamp(year, month, day, hours, minutes, seconds, nanos, scale);
```

The JDBC protocol returns the DATETIME and TIMESTAMP in a string bytes stream. For DATETIME since there is no conversion, the driver receives a string like this:

```
2021-01-26 01:00:00
```

So, nothing could go wrong for DATETIME. However, for TIMESTAMP, remember the SQL test we made at the beginning of this article? Since the driver receives a string byte stream with a time zone converted like this:

```
2021-01-26 02:00:00
```

Then we calculate the UTC epoch at the application side with a Calendar class in JDBC driver:

```java
   this.cal.set(its.getYear(), its.getMonth() - 1, its.getDay(), its.getHours(), its.getMinutes(), its.getSeconds());
   Timestamp ts = new Timestamp(this.cal.getTimeInMillis());
```

So this is where things are wrong, when getTimestamp(i) on TIMESTAMP field, and you change your time zone and read it again, the UTC Epoch will be wrong. The root cause is TIMESTAMP should be designed to return a string including time zone information like '2021-01-26T01:00:00+2' or just return a UTC epoch. The current design of TIMESTAMP in MySQL is totally wrong and very confusing.

Since TIMESTAMP is not implemented correctly and it cannot represent time beyond the year 2038 due to its internal 4 bytes storage design (wrong design again).

Recap, if you want explicit dates in MySQL, you can always convert your local time to UTC and save it to DATETIME field, then you always read the time in UTC, return that information to the frontend application in either UTC epoch, or a string including timezone info. If you want floating time, just use LocalDateTime on DATETIME field.

# Conversion between Application and Browser or Client

Now you should understand, for explicit times your API must have all date fields as UTC epoch time (long) or a string with time zone information, for floating time you should use LocalDateTime.

```
GET /schedule/1
{
  "time1": 1611652827002,             ---- explicit
  "time2": "2021-01-26T01:00:00+2",   ---- explicit
  "time3": "2021-01-26T01:00:00",     ---- floating
}
```
In Java, you can use Long or OffsetDateTime to represent explicit time and LocalDateTime to represent floating time.
```

public class Schedule {
  Long time1;               // explicit
  OffsetDateTime time2;     // explicit
  LocalDateTime time3;      // floating
}
```

If you are using Jackson as your serialization framework with spring, you probably need to register JavaTimeModule to support OffsetDateTime and LocalDateTime.
```
ObjectMapper mapper = new ObjectMapper()
   .registerModule(new Jdk8Module())
   .registerModule(new JavaTimeModule());
```

# Conclusion
To design an application for global users like facebook, you need to educate your developers the concept of "floating" and "explicit" time, use the strategies we discussed in this article to storage and convert dates. In the next article, we will discuss some interesting topics like time scale, clock, and leap second handling, stay tuned.

