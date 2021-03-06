cron
====

A cron library for Go.

## Usage

Callers may register Funcs to be invoked on a given schedule.  Cron will run
them in their own goroutines.

```go
c := new(Cron)
c.Add("0 5 * * * *", func() { fmt.Println("Every 5 minutes") })
c.Add("@hourly", func() { fmt.Println("Every hour") })
c.Start()
..
// Funcs are invoked in their own goroutine, asynchronously.
...
// Funcs may also be added to a running Cron
c.Add("@daily, func() { fmt.Println("Every day") })
..
c.Stop()  // Stop the scheduler (does not stop any jobs already running).
```

## CRON Expression

This section describes the specific format accepted by this cron.  Some snippets
are taken from [the wikipedia article](http://en.wikipedia.org/wiki/Cron).

### Format

A cron expression represents a set of times, using 6 space-separated fields.

Field name | Mandatory? | Allowed values | Allowed special characters
---------- | ---------- | -------------- | --------------------------
Seconds | Yes 	| 0-59 | * / , -
Minutes | Yes 	| 0-59 | * / , -
Hours 	| Yes 	| 0-23 | * / , -
Day of month | Yes | 1-31 | * / , - ?
Month 	| Yes 	| 1-12 or JAN-DEC | * / , -
Day of week | Yes | 0-6 or SUN-SAT | * / , - ?

Note: Month and Day-of-week field values are case insensitive.  "SUN", "Sun",
and "sun" are equally accepted.

### Special Characters

#### Asterisk ( * )

The asterisk indicates that the cron expression will match for all values of the
field; e.g., using an asterisk in the 5th field (month) would indicate every
month.

#### Slash ( / )

Slashes are used to describe increments of ranges. For example 3-59/15 in the
1st field (minutes) would indicate the 3rd minute of the hour and every 15
minutes thereafter. The form "*/..." is equivalent to the form "first-last/...",
that is, an increment over the largest possible range of the field.  The form
"N/..." is accepted as meaning "N-MAX/...", that is, starting at N, use the
increment until the end of that specific range.  It does not wrap around.

#### Comma ( , )

Commas are used to separate items of a list. For example, using "MON,WED,FRI" in
the 5th field (day of week) would mean Mondays, Wednesdays and Fridays.

#### Hyphen ( - )

Hyphens are used to define ranges. For example, 9-17 would indicate every
hour between 9am and 5pm inclusive.

#### Question mark ( ? )

Question mark may be used instead of '*' for leaving either day-of-month or
day-of-week blank.

### Predefined schedules

You may use one of several pre-defined schedules in place of a cron expression.

Entry | Description | Equivalent To
----- | ----------- | -------------
@yearly (or @annually) | Run once a year, midnight, Jan. 1st | <code>0 0 0 1 1 *</code>
@monthly | Run once a month, midnight, first of month | <code>0 0 0 1 * *</code>
@weekly | Run once a week, midnight on Sunday | <code>0 0 0 * * 0</code>
@daily (or @midnight) | Run once a day, midnight | <code>0 0 0 * * *</code>
@hourly | Run once an hour, beginning of hour | <code>0 0 * * * *</code>

## Time zones

All interpretation and scheduling is done in the machine's local time zone (as
provided by the [Go time package](http://www.golang.org/pkg/time)).

Be aware that jobs scheduled during daylight-savings transitions will not be
run!


## Implementation

Cron entries are stored in an array, sorted by their next activation time.  Cron
sleeps until the next job is due to be run.

Upon waking:
* it runs each entry that is active on that second
* it calculates the next run times for the jobs that were run
* it re-sorts the array of entries by next activation time.
* it goes to sleep until the soonest job.
