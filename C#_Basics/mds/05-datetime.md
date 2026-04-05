# C# DateTime and Time Handling

## Table of Contents
1. [DateTime Basics](#datetime-basics)
2. [DateTime Immutability](#datetime-immutability)
3. [DateTime vs DateTimeOffset vs DateOnly vs TimeOnly](#datetime-datetimeoffset-dateonly-timeonly)

---

## DateTime Basics

### Overview
`DateTime` represents a date and time value, typically expressed as a date and time of day.

### Creating DateTime
```csharp
// Current date/time
DateTime now = DateTime.Now;           // Local time
DateTime utcNow = DateTime.UtcNow;    // UTC time

// Specific date
DateTime birthday = new DateTime(1990, 6, 15);
DateTime meeting = new DateTime(2024, 6, 15, 14, 30, 0);

// Parsing
DateTime parsed = DateTime.Parse("2024-06-15");
DateTime.TryParse("15/06/2024", out DateTime result);

// DateTimeKind
DateTime unspecified = new DateTime(2024, 6, 15);
DateTime local = new DateTime(2024, 6, 15, 0, 0, 0, DateTimeKind.Local);
DateTime utc = new DateTime(2024, 6, 15, 0, 0, 0, DateTimeKind.Utc);
```

**Memory Visualization:**
```
DateTime structure (8 bytes):
┌─────────────────────────────────────────────────┐
│ Ticks (long): 638,540,000,000,000,000          │
│   └─ 100-nanosecond units since 1/1/0001        │
│ DateData (ulong): encodes ticks + kind          │
│ ├─ Bits 0-61: Ticks                           │
│ └─ Bits 62-63: DateTimeKind                   │
└─────────────────────────────────────────────────┘

Kind values:
0 = Unspecified
1 = UTC
2 = Local
```

### DateTimeKind
```csharp
// Kind affects interpretation during conversion
DateTime local = DateTime.Now;
Console.WriteLine(local.Kind);        // Local

DateTime utc = DateTime.UtcNow;
Console.WriteLine(utc.Kind);          // Utc

DateTime unspecified = new DateTime(2024, 6, 15);
Console.WriteLine(unspecified.Kind);  // Unspecified

// Converting
DateTime utcFromLocal = local.ToUniversalTime();
DateTime localFromUtc = utc.ToLocalTime();
```

**Real-World Example:**
```csharp
public class EventScheduler
{
    public DateTime ScheduleEvent(string dateInput, string timeInput)
    {
        // Parse user input
        if (DateTime.TryParse(dateInput, out DateTime date))
        {
            var time = TimeSpan.Parse(timeInput);
            
            // Combine date and time
            DateTime eventDateTime = date.Date + time;
            
            // Convert to UTC for storage
            return eventDateTime.ToUniversalTime();
        }
        throw new ArgumentException("Invalid date");
    }
}
```

---

## DateTime Immutability

### Overview
`DateTime` is a value type that cannot be modified after creation. All operations return new instances.

### How Add Methods Work
```csharp
DateTime appointment = new DateTime(2024, 6, 15, 10, 0, 0);

// Original unchanged
DateTime reminder = appointment.AddDays(-1);  // New instance
DateTime end = appointment.AddHours(2);       // New instance

Console.WriteLine(appointment);  // 6/15/2024 10:00:00 AM
Console.WriteLine(reminder);     // 6/14/2024 10:00:00 AM
Console.WriteLine(end);        // 6/15/2024 12:00:00 PM
```

**Memory Visualization:**
```
Original DateTime:
┌─────────────────────────────────────────────────┐
│ appointment: DateTime                           │
│ Ticks: 638,540,000,000,000,000                 │
│ Value: 2024-06-15 10:00:00                     │
└─────────────────────────────────────────────────┘

After AddDays(-1):
┌─────────────────────────────────────────────────┐
│ appointment: (unchanged)                         │
│   Ticks: 638,540,000,000,000,000               │
├─────────────────────────────────────────────────┤
│ reminder: NEW DateTime                          │
│   Ticks: 638,539,136,000,000,000               │
│   Value: 2024-06-14 10:00:00                   │
└─────────────────────────────────────────────────┘

Two separate values on stack (value types)
```

### Common DateTime Methods
```csharp
DateTime now = DateTime.Now;

// Addition (returns new DateTime)
DateTime tomorrow = now.AddDays(1);
DateTime nextWeek = now.AddDays(7);
DateTime nextMonth = now.AddMonths(1);
DateTime nextYear = now.AddYears(1);

// Subtraction (returns TimeSpan)
TimeSpan duration = tomorrow - now;  // 1 day
TimeSpan elapsed = DateTime.Now - startTime;

// Properties (readonly access)
DateTime date = now.Date;           // Midnight of today
int year = now.Year;                // 2024
int month = now.Month;              // 6
int day = now.Day;                  // 15
DayOfWeek dow = now.DayOfWeek;      // Saturday

// Formatting
string iso = now.ToString("yyyy-MM-dd HH:mm:ss");  // 2024-06-15 14:30:00
string custom = now.ToString("dddd, MMMM d, yyyy"); // Saturday, June 15, 2024
```

**Real-World Example:**
```csharp
public class SubscriptionService
{
    public DateTime CalculateExpiry(DateTime startDate, int months)
    {
        // Immutable operations chain
        return startDate
            .AddMonths(months)           // Add subscription duration
            .AddDays(-1)                  // End of previous day
            .Date                         // Reset to midnight
            .AddHours(23).AddMinutes(59); // 11:59 PM
    }
    
    public bool IsExpired(DateTime expiryDate)
    {
        return DateTime.UtcNow > expiryDate.ToUniversalTime();
    }
}

// Usage
var service = new SubscriptionService();
var expiry = service.CalculateExpiry(DateTime.Now, 12);  // 12 months
// Returns: 2025-06-14 23:59:00 (end of day before anniversary)
```

---

## DateTime vs DateTimeOffset vs DateOnly vs TimeOnly

### Overview
.NET provides multiple types for different date/time scenarios.

### Comparison Table

| Type | Stores | Best For |
|------|--------|----------|
| `DateTime` | Local or UTC timestamp | Legacy code, simple cases |
| `DateTimeOffset` | UTC timestamp + offset | APIs, cross-timezone |
| `DateOnly` | Date only (no time) | Birthdays, anniversaries |
| `TimeOnly` | Time only (no date) | Store hours, schedules |

### DateTime vs DateTimeOffset

**DateTime Problem:**
```csharp
// Two developers in different time zones
DateTime meeting = DateTime.Parse("2024-06-15 14:00");
// Dev in NYC: thinks 2 PM EST
// Dev in London: thinks 2 PM GMT
// Actual meeting time: ambiguous!
```

**DateTimeOffset Solution:**
```csharp
// Explicit offset included
DateTimeOffset meeting = new DateTimeOffset(2024, 6, 15, 14, 0, 0, TimeSpan.FromHours(-4)); // EST
// Same moment in time, regardless of viewer's timezone

// Convert to any timezone
TimeZoneInfo pst = TimeZoneInfo.FindSystemTimeZoneById("Pacific Standard Time");
DateTimeOffset pstTime = TimeZoneInfo.ConvertTime(meeting, pst);
```

**Memory Visualization:**
```
DateTimeOffset (10 bytes):
┌─────────────────────────────────────────────────┐
│ DateTime (8 bytes): ticks + kind                │
├─────────────────────────────────────────────────┤
│ Offset (2 bytes): minutes from UTC              │
│ Example: -240 = EST (-4 hours)                  │
└─────────────────────────────────────────────────┘

Comparison:
DateTime: "2024-06-15 14:00:00" (context unknown)
DateTimeOffset: "2024-06-15 14:00:00 -04:00" (explicit!)
```

### DateOnly and TimeOnly (.NET 6+)
```csharp
// DateOnly - calendar date without time
DateOnly birthday = new DateOnly(1990, 6, 15);
DateOnly today = DateOnly.FromDateTime(DateTime.Now);

// Operations
DateOnly nextYear = birthday.AddYears(1);
int age = today.Year - birthday.Year;

// TimeOnly - time of day without date
TimeOnly openingTime = new TimeOnly(9, 0);    // 9:00 AM
TimeOnly closingTime = new TimeOnly(17, 30);  // 5:30 PM

// Operations
bool isOpen = TimeOnly.Now.IsBetween(openingTime, closingTime);
TimeOnly reminder = openingTime.AddMinutes(-30);  // 8:30 AM
```

**Memory Visualization:**
```
DateOnly (4 bytes):
┌─────────────────────────────────────────────────┐
│ DayNumber (int): days since 1/1/0001            │
│ Example: 738,300 = June 15, 2024                │
└─────────────────────────────────────────────────┘

TimeOnly (4 bytes):
┌─────────────────────────────────────────────────┐
│ Ticks (long - packed): time of day              │
│ Example: 32,400,000,000,000 = 9:00:00           │
└─────────────────────────────────────────────────┘

Benefits over DateTime:
- No timezone ambiguity
- Smaller memory footprint
- Clearer intent
```

### When to Use Each

```csharp
// DateOnly: Birthdays, holidays
public class Person
{
    public DateOnly DateOfBirth { get; set; }  // No time component needed
    
    public int CalculateAge(DateOnly asOf)
    {
        return asOf.Year - DateOfBirth.Year -
            (asOf.Month < DateOfBirth.Month ||
             (asOf.Month == DateOfBirth.Month && asOf.Day < DateOfBirth.Day) ? 1 : 0);
    }
}

// TimeOnly: Business hours, schedules
public class BusinessHours
{
    public TimeOnly OpenTime { get; set; }
    public TimeOnly CloseTime { get; set; }
    
    public bool IsOpen(TimeOnly time) => time.IsBetween(OpenTime, CloseTime);
}

// DateTimeOffset: API timestamps, events
public class Event
{
    public DateTimeOffset StartTime { get; set; }  // Universal moment
    public string TimeZoneId { get; set; }         // For display
    
    public DateTimeOffset GetLocalTime(string viewerTimeZone) => 
        TimeZoneInfo.ConvertTime(StartTime, TimeZoneInfo.FindSystemTimeZoneById(viewerTimeZone));
}

// DateTime: Legacy interop, simple local calculations
public class LegacySystem
{
    public DateTime CreatedAt { get; set; }  // Keep for existing databases
}
```

**Real-World Example:**
```csharp
public class GlobalMeetingScheduler
{
    public DateTimeOffset ScheduleMeeting(
        DateOnly date,
        TimeOnly time,
        string organizerTimeZoneId)
    {
        // Combine date, time, and timezone
        var organizerZone = TimeZoneInfo.FindSystemTimeZoneById(organizerTimeZoneId);
        
        // Create DateTime with organizer's local time
        DateTime localDateTime = date.ToDateTime(time);
        
        // Convert to DateTimeOffset (preserves the moment)
        DateTimeOffset meetingTime = new DateTimeOffset(
            localDateTime, 
            organizerZone.GetUtcOffset(localDateTime));
        
        return meetingTime;
    }
    
    public string FormatForAttendee(DateTimeOffset meetingTime, string attendeeTimeZoneId)
    {
        var attendeeZone = TimeZoneInfo.FindSystemTimeZoneById(attendeeTimeZoneId);
        var localTime = TimeZoneInfo.ConvertTime(meetingTime, attendeeZone);
        
        return $"Meeting at {localTime:HH:mm} ({attendeeZone.StandardName})";
    }
}

// Usage
var scheduler = new GlobalMeetingScheduler();
var meeting = scheduler.ScheduleMeeting(
    new DateOnly(2024, 6, 15), 
    new TimeOnly(14, 0), 
    "Eastern Standard Time");

Console.WriteLine(scheduler.FormatForAttendee(meeting, "Pacific Standard Time"));
// Output: "Meeting at 11:00 (Pacific Standard Time)"
```

---

*Source: .NET DateTime documentation, Noda Time best practices, and timezone handling guides.*
