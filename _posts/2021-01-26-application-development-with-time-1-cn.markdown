---
layout: post
title:  "国际化应用开发中的时间处理 - Floating and Explicit Time（中文）"
subtitle: "时区，浮动时间和固定时间"
date:   2021-01-28 17:20:00
published: true
---

# 概述

在为世界各地的用户开发应用程序时，通常会错误地实现时间，这是因为大多数开发人员都不了解浮动时间（Floating Time）和显式时间（Explicit Time）的概念。在本文中，我们将讨论它们是什么以及如何在时区中使用它们。


# 浮动时间和显式时间

“浮动日期/时间”是与特定时区无关的时间值。例如生日。您的生日是在中国的1月1日，如果您移居美国，您的生日仍然是1月1日，但实际上在中国过生日，你会比在美国早12-14个小时。因为生日是个Floating Time，所以我们用日期字符串“ 2020-1-1”来表示生日。另一个例子是圣诞节，“ 2021-12-25T00:00”在不同时区的发生方式有所不同。

“Explicit Time”或“Non-floating Time”具有时区偏移量信息，它表示实际时刻，无论您处于哪个时区都仅发生一次。要实现这一点，您可以将所有明确的日期都用UTC epoch表达，或在您的日期类型中包含明确的时区信息。例如，ISO8601日期字符串，例如“ 2021-01-26T01:00:00Z + 08”。在Java中，时区有两个类似的类：ZoneId和ZoneOffset。 ZoneId的名称类似于“Asia/Beijing”，具有DST支持，可以根据JDK的更新调整DST。 ZoneOffset更为简单，它只是一个类似于数字的偏移量，例如'+08'。相应地，Java中有两个DateTime：OffsetDateTime和ZonedDateTime。稍后我们将详细讨论它们。

显然，如果将它们混合在一起，将火箭发射时间存储在浮动日期时间字段中，则可能会提前8小时发射火箭。这就是为什么您会看到宇航员总是在电影中说休斯顿时间，而他们想要明确的时间。您可能永远不会说自己的生日是在柏林时间，因为它是浮动时间，如果您飞往不同时区的另一个国家，您将不会在几小时前调整生日。

浮动时间和明确时间之间可能会有转换。例如，要在生日当天（浮动时间）发送祝贺电子邮件，您的应用程序可以检测到用户所在的时区，然后计算出明确的发送时间。另一个示例是向不同时区的人发送会议请求，那么它必须是一个明确的时间，但是您需要将其转换为每个参与者在前端的本地时间（浮动）。在本文中，我们主要关注如何正确处理显式时间，因为这比处理浮动时间要困难得多。

只有在

1）您将其正确存储在数据库或缓存中，

2）您在存储与应用程序之间进行了正确的转换，

3）您的应用程序与客户端（如浏览器或操作系统）之间的正确转换后，

才可以正确处理和显示显式时间Android设备。在本节中，我们将讨论您需要注意的三个项目。


# 数据库存储

我们可以拿最流行的MySQL来示范。MySQL里有两个时间类型。

* DATETIME: Eight bytes: A four-byte integer for date packed as YYYY×10000 + MM×100 + DD and a four-byte integer for time packed as HH×10000 + MM×100 + SS
* TIMESTAMP: A four-byte integer representing seconds UTC since the epoch ('1970-01-01 00:00:00' UTC)

在MySQL中，DATETIME是floating日期类型，TIMESTAMP是explicit日期类型，它将在保存时将时间转换为UTC，在加载时将转换回您的本地时间。 听起来不错，但是他实现有错误，主要问题是当您从数据库中检索一行时，返回的时间字符串中没有时区。 我将通过一些测试来更详细地解释它。

现在，让我们进行一些测试以测试这两种日期类型。

首先，我们创建一个包含datetime和timestamp列的表。

```sql
CREATE TABLE `test_date` (
  `C1` bigint DEFAULT NULL,
  `C2` datetime DEFAULT NULL,
  `C3` timestamp NULL DEFAULT NULL
) ENGINE=InnoDB;

```

你可以在MySQL server端设置默认的时区，这样默认情况下所有的客户端链接不指定时区的话都会使用这个默认时区。.
```mysql
[mysqld]
default-time-zone="+10:00"
```
如果要指定时区，客户端可以这样做：
```sql
SET @@time_zone = '+10:00'
```

现在我们切换到 UTC+2, 插入一条记录然后切换到 UTC+3 再去读取它.
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
您可以看到DATETIME没有时区的概念，因此无论客户端设置什么时区，它都将返回相同的字符串“ 2021-01-26 01:00:00”。 而TIMESTAMP在插入时会意识到时区，它将时间字符串转换为UTC Epoch并将其存储在数据库中。 稍后，当您在新的时区中将其取回时，它将UTC Epoch转换为新的时区，作为新的字符串加上一小时。

因此，使用TIMESTAMP似乎是个好主意，因为它明确用于火箭发射，对吧？ 实际上没有，因为从数据库中检索时，TIMESTAMP没有返回时区信息，MySQL的设计是有问题的。好的设计应该是要在结尾带上时区，如下所示：
```
2021-01-26 01:00:00+2
```
或仅返回UTC Epoch，尽管人类很难阅读。 JDBC驱动程序可以通过getTimestamp（index）来获取UTC Epoch时间，但是不幸的是，这在JDBC中也是错误实现的，我们将在下一节中讨论它。

如果从MySQL实例导出数据并将其导入到另一个实例，则无论服务器位于哪个时区，DATETIME字段都不会更改，而TIMESTAMP字段会更改。 如果您将所有日期字段都设置为DATETIME并将UTC Epoch保存到其中，则您的应用程序将正常运行。
# 数据库和应用之间的转换

到目前为止，在JDK8之后，在Java应用程序中建议使用OffsetDateTime和ZonedDateTime作为显式日期类型的日期类型，并且可以将它们用于证券交易所，火箭发射时间表等； LocalDateTime是用于浮动时间的新类，它在内部没有时区信息，您可以将其用于酒店早上呼叫，生日祝贺电子邮件等。

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
你可以看到这两个 ZonedDateTime 其实底层是一样的 UTC Epoch, 他们只是打印的时候按照内部保存的时区信息进行格式化了。

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
您可以看到两个OffsetDateTime实例的行为与ZonedDateTime完全相同。 与ZoneDateTime的唯一区别是，前者使用名为ZoneId的名称（并且可能使用DST），而后者使用ZoneOffset表示时区。

不幸的是，MySQL JDBC驱动程序不支持OffsetDateTime / ZonedDateTime，因为它们是explicit time类型，而DATETIME是floating time类型，其中没有时区信息，二者是无法转换的。 怎么办呢？其实对于具有不同时区的世界各地用户的应用程序，您可以（必须）使用DATETIME保存/加载UTC Epoch来模拟explicit time。 您可能需要历史悠久的Date与MySQL进行交互，因为它实际上是一个明确的UTC Epoch。 对于floating time，请使用LocalDateTime。 下表显示，您只能在Java和MySQL交互中使用前两个日期类型。

| Java Date Type | Explicit or Floating   | Save | Load |
| -------------- | ---------------------- | ---- | ---- |
| Date/Timestamp | Explicit as Epoch      | Y    | Y    |
| LocalDateTIme  | Floating               | Y    | Y    |
| OffsetDateTime | Explicit with TZ/Epoch | N    | N    |
| ZonedDateTime  | Explicit with TZ/Epoch | N    | N    |


下面是使用JDBC处理explicit time的例子:
```java
pst.setTimestamp(2, new Timestamp(System.currentTimeMillis())); // correct UTC epoch
pst.setTimestamp(3, new Timestamp(utcEpochFromFrontEnd));       // correct UTC epoch
pst.setTimestamp(2, new Timestamp(parse("2021-01-01T01:00+2")));// should be able to convert to correct UTC epoch with "+2"
```
上面的示例适用于explicit time，因为Timestamp实例是正确的UTC Epoch时间戳。 但是下面的代码不起作用，因为函数parse（...）不知道此字符串是否在当前时区或UTC，因此很可能无法将其转换为正确的UTC Epoch时间。 请记住，前端应用程序应始终以Long类型提交UTC Epoch时间戳，或显式带有时区的ISO8601日期字符串。
```java
pst.setTimestamp(2, new Timestamp(parse("2021-01-01T01:00"))); // this won't work for explicit time
```
如果你要处理floating time，那么直接可以在DATETIME字段上用LocalDateTime:
```java
pst.setObject(2, LocalDateTime.now());
pst.setString(2, "2021-01-26T01:00:00"); // a string without time zone information
```

# Let's test

现在，让我们编写一个简单的Java应用程序来做两件事：首先，将一些数据保存到我们刚刚创建的表中，然后将会话设置为不同的时区，然后将其读回。

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

从输出中，您可以看到DATETIME可以正常工作，并使用UTC保存和加载，可以将其用作显式时间。 但是TIMESTAMP没有。

首先，TIMESTAMP返回的LocalDateTime调整了一个小时，这是TIMESTAMP在MySQL中的预期行为，但这是没有用的，这是MySQL的错误设计。 这不是浮动时间，也不是明确的时间。 对于浮动日期，不应调整小时。 对于明确的时间，我们应该在LocalDateTime中具有时区信息，但是此类没有时区。

此外，getTimestamp（i）返回的UTC Epoch是错误的。 由于UTC Epoch是0时区（旧称格林威治时间），无论客户端处于哪个时区，我们都应该始终看到相同的UTC Epoch，但是在我们的测试中，您会看到该时间戳时间从1611655784000更改为1611659384000。

这是为什么？ 为什么TIMESTAMP的UTC Epoch是错误的？ 让我们看看JDBC驱动程序中的类MysqlTextValueDecoder。

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
JDBC协议以字符串流返回DATETIME和TIMESTAMP。 对于DATETIME，因为没有任何转换，驱动程序会收到如下字符串：
```
2021-01-26 01:00:00
```

因此，DATETIME不会出错。 但是，对于TIMESTAMP，还记得我们在本文开头进行的SQL测试吗？ 由于驱动程序接收到的字符串字节流的时区转换如下：

```
2021-01-26 02:00:00
```

然后，我们使用JDBC驱动程序中的Calendar类在应用程序端计算UTC时期：

```java
   this.cal.set(its.getYear(), its.getMonth() - 1, its.getDay(), its.getHours(), its.getMinutes(), its.getSeconds());
   Timestamp ts = new Timestamp(this.cal.getTimeInMillis());
```
所以这就是问题所在，当在TIMESTAMP字段上使用getTimestamp(i)并更改时区并再次读取时，UTC Epoch将是错误的。 根本原因是TIMESTAMP应该设计为返回包含时区信息的字符串，例如“ 2021-01-26T01:00:00+2”，或者仅返回UTC Epoch。 MySQL中TIMESTAMP的当前设计是完全错误的并且非常令人困惑。

由于TIMESTAMP的实现不正确，并且由于其内部4字节存储设计（也是错误的设计），它无法表示2038年以后的时间。 

回顾一下，如果您想在MySQL中使用explicit time，则始终可以将本地时间转换为UTC并将其保存到DATETIME字段，然后始终以UTC读取时间，然后以UTC Epoch或包含时区的字符串将该信息返回给前端应用程序。 如果要floating time，只需在DATETIME字段上使用LocalDateTime。

# 应用服务器和前端之间转换

现在您应该了解，对于显式时间，API必须具有所有日期字段（如UTC Epoch时间（长整数）或带有时区信息的字符串），对于浮动时间，应使用LocalDateTime。

```
GET /schedule/1
{
  "time1": 1611652827002,             ---- explicit
  "time2": "2021-01-26T01:00:00+2",   ---- explicit
  "time3": "2021-01-26T01:00:00",     ---- floating
}
```
Java中REST API你可以用Long或者OffsetDateTime来表达 explicit time，用 LocalDateTime 来表达 floating time.
```

public class Schedule {
  Long time1;               // explicit
  OffsetDateTime time2;     // explicit
  LocalDateTime time3;      // floating
}
```

如果将spring用作Jackson的序列化框架，则可能需要注册JavaTimeModule以支持OffsetDateTime和LocalDateTime。
```
ObjectMapper mapper = new ObjectMapper()
   .registerModule(new Jdk8Module())
   .registerModule(new JavaTimeModule());
```

# 结论
若要为诸如Facebook之类的全球用户设计应用程序，您需要教育开发人员“浮动”和“显式”时间的概念，使用我们在本文中讨论的策略来存储和转换日期。 在下一篇文章中，我们将讨论一些有趣的主题，例如Time Scale，时钟和leap second处理，敬请期待。


