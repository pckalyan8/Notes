# 4.4 Date & Time API (Java 8+)

## What & Why

Java's original `java.util.Date` and `java.util.Calendar` classes were notoriously painful to work with. `Date` was mutable (a thread-safety disaster), it represented an instant but its getter/setter methods used year offsets from 1900, month indices starting at 0, and it mixed date and time concerns with timezone concerns in a single object. `Calendar` tried to fix things but introduced its own complexity and inconsistency.

Java 8 replaced all of this with `java.time`, a well-designed API inspired by the Joda-Time library. The core principles are **immutability** (all objects are immutable and therefore thread-safe), **clear semantics** (each class represents one precisely-defined concept), and **fluent style** (operations return new instances rather than mutating the current one).

---

## 1. The Core Classes at a Glance

Before diving in, it helps to understand what each class *represents*:

`LocalDate` — a date (year, month, day) with no time and no timezone. Think "2025-12-25".  
`LocalTime` — a time of day with no date and no timezone. Think "14:30:00".  
`LocalDateTime` — a date combined with a time, but still no timezone. Think "2025-12-25T14:30:00".  
`ZonedDateTime` — a full date-time with a timezone. Think "2025-12-25T14:30:00+09:00[Asia/Tokyo]".  
`Instant` — a point on the machine timeline, stored as seconds since the Unix epoch (1970-01-01T00:00:00Z). This is what you use for "right now" in logs and databases.  
`Duration` — a time-based amount (hours, minutes, seconds, nanoseconds). "3 hours and 45 minutes."  
`Period` — a date-based amount (years, months, days). "2 years, 3 months, and 10 days."

---

## 2. LocalDate

```java
import java.time.LocalDate;
import java.time.Month;
import java.time.DayOfWeek;

// Creating instances
LocalDate today      = LocalDate.now();               // current date in system timezone
LocalDate christmas  = LocalDate.of(2025, 12, 25);    // explicit date
LocalDate fromString = LocalDate.parse("2025-03-15"); // ISO-8601 format (default)

// Reading components
System.out.println(christmas.getYear());        // 2025
System.out.println(christmas.getMonth());       // DECEMBER (enum)
System.out.println(christmas.getMonthValue());  // 12 (int)
System.out.println(christmas.getDayOfMonth());  // 25
System.out.println(christmas.getDayOfWeek());   // THURSDAY (enum)

// Arithmetic — all operations return a NEW LocalDate (immutable)
LocalDate nextWeek       = today.plusWeeks(1);
LocalDate lastYear       = today.minusYears(1);
LocalDate startOfMonth   = today.withDayOfMonth(1);
LocalDate endOfYear      = today.withDayOfYear(today.lengthOfYear());

// Comparison
System.out.println(christmas.isAfter(today));   // true (if run before Christmas 2025)
System.out.println(christmas.isBefore(today));  // false
System.out.println(today.isEqual(LocalDate.now())); // true

// Useful queries
System.out.println(christmas.isLeapYear());     // false (2025 not a leap year)
System.out.println(christmas.lengthOfMonth());  // 31 (days in December)

// Check the day of week
if (today.getDayOfWeek() == DayOfWeek.SATURDAY ||
    today.getDayOfWeek() == DayOfWeek.SUNDAY) {
    System.out.println("It's the weekend!");
}
```

---

## 3. LocalTime and LocalDateTime

```java
import java.time.LocalTime;
import java.time.LocalDateTime;

// LocalTime
LocalTime now          = LocalTime.now();
LocalTime meetingTime  = LocalTime.of(14, 30, 0);           // 14:30:00
LocalTime fromString   = LocalTime.parse("09:15:30");

System.out.println(meetingTime.getHour());   // 14
System.out.println(meetingTime.getMinute()); // 30

LocalTime oneHourLater = meetingTime.plusHours(1);    // 15:30:00
LocalTime truncated    = now.truncatedTo(ChronoUnit.MINUTES); // drops seconds/nanos

// LocalDateTime = LocalDate + LocalTime
LocalDateTime dateTime = LocalDateTime.of(2025, 6, 15, 10, 30, 0);
LocalDateTime also     = LocalDate.of(2025, 6, 15).atTime(10, 30);
LocalDateTime now2     = LocalDateTime.now();

// Extract components
LocalDate datePart = dateTime.toLocalDate();
LocalTime timePart = dateTime.toLocalTime();
```

---

## 4. ZonedDateTime and ZoneId

When you need to account for timezones (scheduling, displaying times to international users, converting stored UTC timestamps to local times), you need `ZonedDateTime`.

```java
import java.time.ZonedDateTime;
import java.time.ZoneId;
import java.time.ZoneOffset;

// Create a ZonedDateTime in a specific timezone
ZonedDateTime tokyoNow  = ZonedDateTime.now(ZoneId.of("Asia/Tokyo"));
ZonedDateTime londonNow = ZonedDateTime.now(ZoneId.of("Europe/London"));
ZonedDateTime utcNow    = ZonedDateTime.now(ZoneOffset.UTC);

// List all available timezone IDs
ZoneId.getAvailableZoneIds().stream()
    .filter(id -> id.startsWith("America/"))
    .sorted()
    .forEach(System.out::println);

// Convert a LocalDateTime to a ZonedDateTime
LocalDateTime local = LocalDateTime.of(2025, 6, 15, 10, 0, 0);
ZonedDateTime zoned = local.atZone(ZoneId.of("America/New_York"));

// Convert between timezones
ZonedDateTime inTokyo = zoned.withZoneSameInstant(ZoneId.of("Asia/Tokyo"));
System.out.println("New York: " + zoned);    // 2025-06-15T10:00-04:00[America/New_York]
System.out.println("Tokyo:    " + inTokyo);  // 2025-06-15T23:00+09:00[Asia/Tokyo]
// Note: the instant is the same — just displayed in a different timezone
```

**Key distinction:** `withZoneSameInstant()` converts to the same moment in a different zone. `withZoneSameLocal()` keeps the same clock time but changes the zone label (different instant).

---

## 5. Instant — Machine Time

`Instant` represents a point on the universal timeline. It's the right type for "when did this event happen?" in logs, audit trails, and databases. Always store in UTC, always display using a timezone.

```java
import java.time.Instant;
import java.time.temporal.ChronoUnit;

Instant now    = Instant.now();                         // current UTC moment
Instant epoch  = Instant.EPOCH;                        // 1970-01-01T00:00:00Z
Instant future = Instant.now().plus(1, ChronoUnit.HOURS);

System.out.println(now.toEpochMilli());                 // milliseconds since epoch
System.out.println(now.getEpochSecond());               // seconds since epoch

// Convert to ZonedDateTime for display
ZonedDateTime display = now.atZone(ZoneId.of("America/Los_Angeles"));

// Convert from legacy Date
java.util.Date legacyDate = new java.util.Date();
Instant fromLegacy = legacyDate.toInstant();

// Convert back to legacy Date (for frameworks that still require it)
java.util.Date backToLegacy = java.util.Date.from(now);
```

---

## 6. Duration and Period

`Duration` measures a time-based amount; `Period` measures a date-based amount. They look similar but operate on different units and serve different purposes.

```java
import java.time.Duration;
import java.time.Period;

// --- Duration: hours, minutes, seconds, nanoseconds ---
Duration twoHours        = Duration.ofHours(2);
Duration ninetyMinutes   = Duration.ofMinutes(90);
Duration halfSecond      = Duration.ofMillis(500);

// Compute the duration between two instants/times
Instant start = Instant.now();
// ... do some work ...
Instant end   = Instant.now();
Duration elapsed = Duration.between(start, end);
System.out.println("Elapsed: " + elapsed.toSeconds() + "s " + elapsed.toMillisPart() + "ms");

LocalTime openTime  = LocalTime.of(9, 0);
LocalTime closeTime = LocalTime.of(17, 30);
Duration workDay    = Duration.between(openTime, closeTime);
System.out.println("Work day: " + workDay.toHours() + " hours"); // 8

// --- Period: years, months, days ---
Period oneYear    = Period.ofYears(1);
Period sixMonths  = Period.ofMonths(6);
Period twoWeeks   = Period.ofWeeks(2);

LocalDate birthday  = LocalDate.of(1990, 4, 15);
LocalDate today     = LocalDate.now();
Period age          = Period.between(birthday, today);
System.out.printf("Age: %d years, %d months, %d days%n",
    age.getYears(), age.getMonths(), age.getDays());

// Adding a Period to a date
LocalDate nextBirthday = birthday.plus(Period.ofYears(age.getYears() + 1));
```

---

## 7. DateTimeFormatter — Formatting and Parsing

```java
import java.time.format.DateTimeFormatter;
import java.util.Locale;

LocalDateTime now = LocalDateTime.now();

// Predefined ISO formatters
System.out.println(now.format(DateTimeFormatter.ISO_LOCAL_DATE));       // 2025-06-15
System.out.println(now.format(DateTimeFormatter.ISO_LOCAL_DATE_TIME)); // 2025-06-15T14:30:00

// Custom patterns
DateTimeFormatter custom = DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm");
System.out.println(now.format(custom)); // 15/06/2025 14:30

// Locale-sensitive formatting
DateTimeFormatter french = DateTimeFormatter.ofPattern("d MMMM yyyy", Locale.FRENCH);
System.out.println(now.format(french)); // 15 juin 2025

// Parsing a string into a date
LocalDate parsed = LocalDate.parse("15/06/2025", DateTimeFormatter.ofPattern("dd/MM/yyyy"));

// DateTimeFormatter is thread-safe — safe to store as a static constant
public static final DateTimeFormatter LOG_FORMAT =
    DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss.SSS");
```

---

## 8. TemporalAdjusters

`TemporalAdjusters` provides factory methods for common calendar adjustments:

```java
import java.time.temporal.TemporalAdjusters;

LocalDate today = LocalDate.now();

LocalDate firstDayOfMonth = today.with(TemporalAdjusters.firstDayOfMonth());
LocalDate lastDayOfMonth  = today.with(TemporalAdjusters.lastDayOfMonth());
LocalDate nextMonday      = today.with(TemporalAdjusters.next(DayOfWeek.MONDAY));
LocalDate lastFriday      = today.with(TemporalAdjusters.previous(DayOfWeek.FRIDAY));
LocalDate firstMondayOfMonth = today.with(TemporalAdjusters.firstInMonth(DayOfWeek.MONDAY));

// Custom adjuster using a lambda
// Find the next working day (skip weekends)
TemporalAdjuster nextWorkingDay = temporal -> {
    LocalDate date = LocalDate.from(temporal);
    DayOfWeek dow  = date.getDayOfWeek();
    int daysToAdd  = (dow == DayOfWeek.FRIDAY) ? 3
                   : (dow == DayOfWeek.SATURDAY) ? 2
                   : 1;
    return date.plusDays(daysToAdd);
};

LocalDate nextWorkDay = today.with(nextWorkingDay);
```

---

## 9. Clock — Controllable Time for Testing

`Clock` provides the current date and time to `now()` methods. By default it uses the system clock, but in tests you can inject a fixed or offset clock to get deterministic behaviour without mocking static methods:

```java
import java.time.Clock;

// Production code — accept a Clock parameter
public class OrderService {
    private final Clock clock;

    public OrderService(Clock clock) { this.clock = clock; }
    public OrderService() { this(Clock.systemUTC()); }

    public Order createOrder() {
        return new Order(Instant.now(clock)); // uses injected clock
    }
}

// In tests — use a fixed clock for deterministic results
Clock fixedClock = Clock.fixed(Instant.parse("2025-01-01T12:00:00Z"), ZoneOffset.UTC);
OrderService service = new OrderService(fixedClock);
Order order = service.createOrder();
// order.getCreatedAt() is guaranteed to be 2025-01-01T12:00:00Z
```

---

## ⚡ Best Practices & Important Points

**Store timestamps as `Instant` in databases and logs.** Store always in UTC, display always with a timezone. Storing a `LocalDateTime` in a database can lead to ambiguity when the server's timezone changes (e.g., daylight saving transitions).

**`LocalDate`, `LocalTime`, and `LocalDateTime` have no timezone.** They are not "implicitly UTC". They're useful for representing dates and times that have no timezone context, like "the system will run maintenance every day at 03:00", regardless of the machine's timezone.

**`DateTimeFormatter` is thread-safe.** Unlike the old `SimpleDateFormat`, you can safely declare a `DateTimeFormatter` as a `static final` constant and share it across threads.

**Never use `java.util.Date` or `java.util.Calendar` in new code.** These classes are legacy. If you're interoperating with older APIs, convert immediately: `date.toInstant()` to enter the modern API, `Date.from(instant)` to go back.

**Use `ChronoUnit` for generic time arithmetic.** `now.plus(3, ChronoUnit.DAYS)` reads more clearly than `now.plusDays(3)` in generic/templated code, and `ChronoUnit.between(start, end)` computes precise differences in any unit.

**`Period` and `Duration` are not interchangeable.** `Duration.between(date1, date2)` doesn't exist for `LocalDate` — use `ChronoUnit.DAYS.between(date1, date2)` for that. `Duration` is strictly for time quantities, `Period` for date quantities.
