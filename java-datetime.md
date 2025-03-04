# Day 9: Java日期和时间处理

## 引言

Java 8引入的新日期时间API（java.time包）提供了一套全面且易用的日期时间处理工具。相比旧的java.util.Date和java.util.Calendar，新API设计得更加合理，使用起来更加方便。本文将详细介绍Java日期和时间处理的核心概念和实践应用。

## 1. 日期时间基础

### 1.1 核心类

```java
// 本地日期
LocalDate date = LocalDate.now();
LocalDate specificDate = LocalDate.of(2023, 12, 25);

// 本地时间
LocalTime time = LocalTime.now();
LocalTime specificTime = LocalTime.of(13, 30, 0);

// 本地日期时间
LocalDateTime dateTime = LocalDateTime.now();
LocalDateTime specificDateTime = LocalDateTime.of(2023, 12, 25, 13, 30, 0);

// 带时区的日期时间
ZonedDateTime zonedDateTime = ZonedDateTime.now();
ZonedDateTime specificZonedDateTime = ZonedDateTime.of(dateTime, ZoneId.of("Asia/Shanghai"));
```

### 1.2 时间段和周期

```java
// 时间段（Duration）用于处理时分秒
Duration duration = Duration.between(time1, time2);
Duration oneHour = Duration.ofHours(1);

// 时期（Period）用于处理年月日
Period period = Period.between(date1, date2);
Period oneYear = Period.ofYears(1);
```

## 2. 日期时间操作

### 2.1 日期计算

```java
public class DateCalculator {
    public static LocalDate calculateFutureDate(LocalDate startDate, int days) {
        return startDate.plusDays(days);
    }
    
    public static long daysBetween(LocalDate date1, LocalDate date2) {
        return ChronoUnit.DAYS.between(date1, date2);
    }
    
    public static LocalDate getLastDayOfMonth(LocalDate date) {
        return date.with(TemporalAdjusters.lastDayOfMonth());
    }
    
    public static LocalDate getNextWorkday(LocalDate date) {
        return date.with(TemporalAdjusters.next(DayOfWeek.MONDAY));
    }
}
```

### 2.2 格式化和解析

```java
public class DateFormatter {
    private static final DateTimeFormatter DATE_FORMATTER = 
        DateTimeFormatter.ofPattern("yyyy-MM-dd");
    private static final DateTimeFormatter TIME_FORMATTER = 
        DateTimeFormatter.ofPattern("HH:mm:ss");
    private static final DateTimeFormatter DATETIME_FORMATTER = 
        DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
    
    public static String formatDate(LocalDate date) {
        return date.format(DATE_FORMATTER);
    }
    
    public static String formatTime(LocalTime time) {
        return time.format(TIME_FORMATTER);
    }
    
    public static String formatDateTime(LocalDateTime dateTime) {
        return dateTime.format(DATETIME_FORMATTER);
    }
    
    public static LocalDate parseDate(String dateStr) {
        return LocalDate.parse(dateStr, DATE_FORMATTER);
    }
    
    public static LocalTime parseTime(String timeStr) {
        return LocalTime.parse(timeStr, TIME_FORMATTER);
    }
    
    public static LocalDateTime parseDateTime(String dateTimeStr) {
        return LocalDateTime.parse(dateTimeStr, DATETIME_FORMATTER);
    }
}
```

## 3. 时区处理

### 3.1 时区转换

```java
public class TimeZoneConverter {
    public static ZonedDateTime convertToZone(LocalDateTime dateTime, 
            String sourceZone, String targetZone) {
        ZoneId sourceZoneId = ZoneId.of(sourceZone);
        ZoneId targetZoneId = ZoneId.of(targetZone);
        
        ZonedDateTime sourceDateTime = dateTime.atZone(sourceZoneId);
        return sourceDateTime.withZoneSameInstant(targetZoneId);
    }
    
    public static LocalDateTime getLocalDateTime(ZonedDateTime zonedDateTime) {
        return zonedDateTime.toLocalDateTime();
    }
    
    public static Instant toInstant(LocalDateTime dateTime, String zoneId) {
        return dateTime.atZone(ZoneId.of(zoneId)).toInstant();
    }
}
```

## 4. 实践案例

### 4.1 日期范围处理

```java
public class DateRangeUtil {
    public static List<LocalDate> getDatesBetween(LocalDate startDate, 
            LocalDate endDate) {
        long numOfDays = ChronoUnit.DAYS.between(startDate, endDate);
        return Stream.iterate(startDate, date -> date.plusDays(1))
                .limit(numOfDays + 1)
                .collect(Collectors.toList());
    }
    
    public static boolean isOverlap(LocalDate start1, LocalDate end1,
            LocalDate start2, LocalDate end2) {
        return !start1.isAfter(end2) && !start2.isAfter(end1);
    }
    
    public static Period getAge(LocalDate birthDate) {
        return Period.between(birthDate, LocalDate.now());
    }
}
```

### 4.2 工作日历

```java
public class BusinessCalendar {
    private Set<LocalDate> holidays;
    
    public BusinessCalendar() {
        this.holidays = new HashSet<>();
    }
    
    public void addHoliday(LocalDate date) {
        holidays.add(date);
    }
    
    public boolean isBusinessDay(LocalDate date) {
        DayOfWeek dayOfWeek = date.getDayOfWeek();
        return !holidays.contains(date) && 
               dayOfWeek != DayOfWeek.SATURDAY && 
               dayOfWeek != DayOfWeek.SUNDAY;
    }
    
    public LocalDate getNextBusinessDay(LocalDate date) {
        LocalDate nextDay = date.plusDays(1);
        while (!isBusinessDay(nextDay)) {
            nextDay = nextDay.plusDays(1);
        }
        return nextDay;
    }
    
    public int getBusinessDaysBetween(LocalDate start, LocalDate end) {
        return (int) start.datesUntil(end.plusDays(1))
                .filter(this::isBusinessDay)
                .count();
    }
}
```

## 5. 最佳实践

1. 优先使用新的日期时间API
2. 正确处理时区
3. 使用合适的日期时间类型
4. 注意日期时间的不可变性
5. 合理使用格式化器

## 总结

本文介绍了Java日期和时间处理的核心概念和实践应用，包括：

1. 日期时间API的基本使用
2. 日期时间的计算和操作
3. 时区处理
4. 格式化和解析
5. 实践案例和最佳实践

通过掌握这些知识，我们可以更好地处理日期时间相关的业务需求，避免常见的日期时间处理错误。

## 参考资源

1. Java官方文档：https://docs.oracle.com/javase/tutorial/datetime/
2. Java 8日期时间API指南
3. 时区处理最佳实践
4. 日期时间格式化指南