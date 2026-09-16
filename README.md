# cron-nv

A **crontab expression** is a line of fields that says when a job should
run. `0 9 * * 1` is nine in the morning, every Monday. The five-field
form is specified by POSIX in
[crontab](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/crontab.html)
and implemented by [Vixie cron](https://man7.org/linux/man-pages/man5/crontab.5.html),
which is the `cron` on most machines; the six- and seven-field forms and
their extra syntax come from
[Quartz](https://www.quartz-scheduler.org/documentation/quartz-2.3.0/tutorials/crontrigger.html).
This package reads those lines, says what one means in English, and
computes when it fires next. It runs no jobs and reads no clock.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What it is

A crontab expression is a whitespace-separated list of **fields**, one
per unit of time, in a fixed order. Each field holds a set of values,
and a job fires at every moment where all the fields match at once.

| Field | Range | In which dialect |
| --- | --- | --- |
| second | 0–59 | six-field and Quartz only |
| minute | 0–59 | always |
| hour | 0–23 | always |
| day of month | 1–31 | always |
| month | 1–12, or `JAN`–`DEC` | always |
| day of week | 0–7, or `SUN`–`SAT` | always |
| year | 1970–2099 | Quartz only, and optional |

A field is written with four pieces of syntax. `*` is every value in the
range. A **list** is values separated by commas: `0,15,30,45`. A
**range** is two values separated by a hyphen and includes both ends:
`MON-FRI`. A **step** is a base and a stride separated by a slash, so
`*/15` is every fifteenth value from the bottom of the range and
`10-50/15` is every fifteenth from 10. There are also seven
**shorthands** that stand for whole expressions: `@yearly`, `@annually`,
`@monthly`, `@weekly`, `@daily`, `@midnight` and `@hourly`.

Quartz adds four pieces of syntax the Unix dialect does not have. `L` in
the day-of-month field is the last day of the month, and `L-3` is three
days before it. `nW` is the weekday nearest the `n`th, never crossing
into another month. `d#n` is the `n`th weekday `d` of the month, so
`5#3` is the third Friday. `?` means "no opinion", and Quartz requires
exactly one of the two day fields to carry it.

The times a crontab expression names are **wall-clock times**: what a
clock on the wall in some place reads, with no offset attached. An
**instant** is a point on the world's timeline, counted here in seconds
since 1970-01-01 00:00:00 UTC. A **time zone** is the rule that converts
between the two, and the rule changes twice a year wherever daylight
saving is observed. So `0 9 * * 1` names the same wall time all year and
a different instant in summer than in winter, and twice a year it names
a wall time that either did not happen or happened twice. Computing the
next fire therefore takes a zone, and the answer says what the clocks
did to it.

Nothing in this package reads a clock, sleeps or runs anything. The
instant to search from is an argument. That is what makes a schedule
testable without waiting for it, and it is why the package declares no
effects.

## Install

```
novo pkg add cron-nv
```

## Example

```novo
use std.str
use cronexpr
use cronnext
use tzdata

// When a crontab line fires next, in a named zone, after an instant.
fn when_next(expr: Str, zone_name: Str, now: Int) -> Str []
    let db = tzdata.data_compact()
    match tzdata.zone(db, zone_name)
        Err(_) => "no such zone"
        Ok(z)  =>
            match cronexpr.parse(expr)
                // The refusal names the field it came from.
                Err(e) => e.message()
                Ok(s)  =>
                    // `default_dst` is Vixie cron's rule for a fire the
                    // clocks moved.
                    let fire = cronnext.next_after(s, z, now, cronnext.default_dst())
                    match cronnext.instant_of(fire)
                        // The schedule has no fire inside the search bound.
                        None     => "never"
                        // `dst_note` is empty unless the clocks moved this one.
                        Some(at) => str.from_int(at) + cronnext.dst_note(fire)

fn main() [io]
    println(when_next("0 9 * * 1", "Europe/Berlin", 1750000000))
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: cron-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `cronexpr` | The whole line: the parsed schedule with its seven fields and the text it was written as, the three ways to parse one, the validator that reports every mistake, and the questions asked about a schedule as a whole. |
| `cronfield` | One column: the three dialects, the terms a field can be written as, the set of values a field admits, and the readers, the parser and the printer for one field. |
| `cronnext` | The arithmetic: the next and previous fire in a zone, the fires in a range, the daylight-saving policies, the answer a fire is reported as, and the same search over civil time with no zone in it. |
| `crondesc` | The schedule in English: the one-line sentence, the labelled pieces it is assembled from, and the names and numbers those pieces are built out of. |
| `cronerr` | The refusals: which of the seven columns a mistake is in, what the mistake was, and the bounds and name of each column. |

## How to choose an entry point

**`cronnext.next_after` is the ordinary call.** Give it a schedule, a
zone, the instant to search from and a daylight-saving policy, and it
answers the one next fire.

**`cronnext.next_n` and `.fires_between` answer many.** `next_n` is what
a "next five runs" panel is drawn from. `fires_between` takes a
half-open range of instants and is what a process that was down asks for
on the way back up. Both keep their place in the search, so they are
cheaper than calling `next_after` in a loop.

**`cronnext.matches_at` asks instead of searching.** A program that
wakes once a minute asks whether the schedule fires at this instant. It
is a test, not a search.

**`cronnext.matches_civil`, `.matches_day`, `.next_civil_after` and
`.prev_civil_before` take no zone at all.** They work in civil time,
which is a date and a time of day with no offset. Take them when you are
already working in one place's wall clock, and to test the field logic
without a time-zone database.

**`cronexpr.check` validates without building anything.** It answers
every refusal a line has rather than the first, which is what a form
that shows a person their mistakes needs. `cronfield.parse_field` is the
same for one box at a time.

## The rules a user needs

1. **When both day fields are restricted, the schedule fires on
   either.** `0 0 13 * 5` is midnight on the 13th *and* midnight on
   every Friday. It is not Friday the 13th. When one of the two is `*`
   it constrains nothing and the other decides alone. This is POSIX's
   rule in `crontab`'s "EXTENDED DESCRIPTION" and Vixie cron's
   behaviour. `cronexpr.fires_on_either_day` answers whether a
   particular line is under it, and `crondesc.day_rule_note` says it in
   the sentence a user interface prints.
2. **The dialect is an argument and is never guessed from the field
   count.** `0 0 1 1 *` is a valid five-field line, meaning midnight on
   New Year's Day, and a valid six-field one, meaning the first second
   of every minute in January. The two are not close. `cronexpr.parse`
   is the five-field Vixie dialect; `cronexpr.parse_with` takes a
   `CronDialect`.

   | `CronDialect` | Fields | Extensions |
   | --- | --- | --- |
   | `CronDialectUnix` | 5 | `@daily` and the other shorthands; no `L`, `W`, `#` or `?` |
   | `CronDialectSixField` | 6 | seconds in front, nothing else |
   | `CronDialectQuartz` | 6 or 7 | `L`, `W`, `#`, `?`, and the year field |

3. **A line using a Quartz extension does nothing in a Unix crontab.**
   It parses here and the machine it is installed on ignores it.
   `cronexpr.uses_extensions` is the question to ask before writing a
   line into `/etc/crontab`.
4. **A next fire cannot be computed by adding a duration to the last
   one.** A weekly job is 604800 seconds later for fifty weeks of the
   year and an hour out for the other two. `cronnext.next_after`
   searches the civil calendar and applies the zone at the end, which is
   why it takes a zone.
5. **A fire can land on a wall time that never happened, or on one that
   happened twice, and the answer says which.** `CronFire` has four
   variants. `CronFireAt` is the ordinary one. `CronFireShifted` carries
   both the wall time that was asked for and the one it will run at.
   `CronFireRepeat` carries whether this is the first or the second
   occurrence. `CronFireNever` is no fire at all. Every variant that
   fires carries the instant, the local wall time and the offset in
   force. `cronnext.instant_of` reads the instant out of any of them,
   `.is_dst_affected` is true for anything but `CronFireAt`, and
   `.dst_note` writes the sentence an operator needs to read.
6. **The daylight-saving policy is an argument, and each value is a real
   implementation's behaviour.**

   | `CronDst` | A wall time that never happened | A wall time that happened twice |
   | --- | --- | --- |
   | `CronDstSkip` | does not fire | fires once |
   | `CronDstRunOnce` | fires at the instant the clock jumped to | fires once |
   | `CronDstRunBoth` | does not fire | fires **twice** |

   `cronnext.default_dst` answers `CronDstRunOnce`, which is Vixie
   cron's rule for a job whose hour and minute are both fixed.
   `CronDstRunBoth` is what a job that reads a meter wants: the repeated
   hour really did happen twice.
7. **Vixie cron's other half is not here, and is named rather than
   approximated.** A job with a wildcard hour, such as `*/10 * * * *`,
   runs once per wall-clock occurrence in Vixie cron: the repeated hour
   fires ten times twice and the skipped hour not at all. That is
   `CronDstRunBoth` and `CronDstSkip` applying at the same time, and
   this interface has no value for the combination.
8. **`CronFireNever` means "not inside the search bound".** The search
   looks `cronnext.search_years()` years forward or back, which is four,
   and then gives up. Two very different schedules answer it: `0 0 30 2
   *` can never fire at all, and a Quartz line whose year field has run
   out will not fire again. `cronexpr.is_satisfiable` tells them apart
   and `cronnext.explain_never` names the field that made it
   impossible.
9. **Both `0` and `7` mean Sunday.** The day-of-week field has eight
   values for seven days, which is why `cronerr.field_bounds` answers
   `(0, 7)` for it rather than `(0, 6)`. Names are accepted in any case,
   because every crontab implementation accepts `mon`, `Mon` and `MON`.
10. **A field keeps the terms it was written as, not only the set they
    denote.** `*/15`, `0,15,30,45` and `0-59/15` match identically, and
    `cronfield.format_field` gives each of them back as it was written.
    That is what lets a tool rewrite one field of a line and leave the
    other four looking like what a person typed, and what lets
    `crondesc.describe` say "every 15 minutes" rather than "at minute 0,
    15, 30 and 45". `cronexpr.equivalent` is the question of whether two
    schedules admit the same times, regardless of spelling.
11. **`L`, `W` and `#` are not in the field's value set.** A term whose
    meaning depends on the month it is evaluated in cannot be a fixed
    set of days: `L` is the 28th, 29th, 30th or 31st, and `5#3` is a
    different date every month. `cronfield.matches` is a test against
    the set alone and answers `false` for those days;
    `cronfield.has_dynamic_terms` is how a caller finds out the set is
    not the whole answer, and `cronnext.matches_day` is the call that
    evaluates them, because it knows which month it is in.
12. **`@reboot` is not accepted, and that is deliberate.**
    `cronexpr.parse_shorthand` takes the seven shorthands that stand for
    a schedule. `@reboot` is an instruction to a daemon about its own
    start-up and has no next fire to compute. A program reading a
    crontab file checks for it before calling.
13. **A range that runs backwards is refused.** `FRI-MON` could mean
    "Friday through Monday the long way round" or "nothing", and both
    readings are wrong for somebody. `FRI-SUN,MON` is the spelling that
    means the first.
14. **`crondesc.cadence` answers an empty string when no short word is
    exactly right.** A job that fires at 09:00 on weekdays is not
    "daily", and a summary that said so is a summary somebody plans a
    deployment around.
15. **Every refusal names its field and, where there is one, its
    offset.** `cronerr.field_of` and `.offset_of` read them back, and
    `.offset_of` answers `-1` for a refusal about the line as a whole.
    `cronexpr.check` answers all of them, because a person fixing three
    mistakes should be told about three.

## What is not included

- **A clock, a daemon and sleeping.** This package computes; it never
  runs anything. Every entry point takes the instant to search from as
  an argument. A program holds its own loop and calls `next_after`.
- **A time-zone database.** [tz-nv](https://novo-lang.org/packages/tz-nv)
  is where a `TzZone` comes from, and this package takes one as an
  argument.
- **`@reboot`.** See rule 12.
- **A language other than English.** `crondesc` writes one language with
  a fixed vocabulary. `crondesc.phrases` answers the labelled pieces the
  sentence is assembled from, one per column, so another language can be
  built from the same parts without parsing anything again.
- **A daylight-saving policy for a wildcard hour.** See rule 7.
- **Running on a microcontroller.** No such claim is made. A schedule's
  arithmetic is calendar-nv's, and calendar-nv makes no such claim
  either.

## Related packages

- [calendar-nv](https://novo-lang.org/packages/calendar-nv) is the civil
  arithmetic underneath: what weekday a date falls on, how many days a
  month has, and the `CivilDate` and `CivilDateTime` that every answer
  here is expressed in.
- [tz-nv](https://novo-lang.org/packages/tz-nv) is the zone: the rule
  that turns a wall time into an instant, and back. It is what makes the
  three daylight-saving answers possible.
- [chrono-nv](https://novo-lang.org/packages/chrono-nv) reads the clock
  and formats an instant. Take it for "what time is it now"; take this
  one for "when does this line fire next".
- [humantime-nv](https://novo-lang.org/packages/humantime-nv) parses and
  prints durations and timestamps the way a person writes them, such as
  `15m` and `2 hours ago`. It is the other half of a configuration file
  that also holds a crontab line.
- [timer-nv](https://novo-lang.org/packages/timer-nv) holds many
  deadlines and tells you which is due. A program that has computed
  fires with this package puts them there.

## Tests

```bash
novo test --isolate tests/cronexpr_tests.nv  # 9 tests: the line, the fields and the refusals
novo test --isolate tests/cronnext_tests.nv  # 9 tests: the next fire and what the clocks do to it
novo test --isolate tests/crondesc_tests.nv  # 7 tests: the sentence
```

The expressions and their answers come from the reference
implementations. croniter's `test_croniter.py` supplies the Unix dialect
and the next-fire search; Quartz's `CronExpressionTest` supplies the
extensions. Where the two disagree, the dialect decides. The
daylight-saving instants are Europe/Berlin's 2026 changes, on the last
Sundays of March and October, as `zdump` prints them.

`cronnext_tests.nv` is mostly the two days a year the wall clock is not
a clock, because a next-fire calculation over a zone with a fixed offset
is arithmetic anyone can write. `cronexpr_tests.nv` asserts the day-field
union rule, the dialect distinctions and that each refusal names its
field. `crondesc_tests.nv` asserts that two schedules which fire
differently are never described the same way, and that a schedule no
short phrase fits is described at length rather than rounded.

No test reads a clock or waits for anything; every assertion names an
instant. The tests compile today and fail at run, each on the
`not implemented: cron-nv.<module>.<fn>` panic that is its body. That is
the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `cronexpr.parse`, `.parse_with`, `.parse_shorthand`, `.is_shorthand`, `.check` | no |
| `cronexpr.format`, `.equivalent`, `.every_minute`, `.field_of` | no |
| `cronexpr.fires_on_either_day`, `.uses_extensions`, `.is_satisfiable` | no |
| `cronfield.parse_field`, `.any_field`, `.format_field` | no |
| `cronfield.matches`, `.values`, `.next_value`, `.prev_value` | no |
| `cronfield.is_wildcard`, `.is_unset`, `.has_dynamic_terms` | no |
| `cronfield.name_of`, `.value_of_name` | no |
| `cronnext.next_after`, `.prev_before`, `.next_n`, `.fires_between`, `.matches_at` | no |
| `cronnext.matches_civil`, `.matches_day`, `.next_civil_after`, `.prev_civil_before` | no |
| `cronnext.default_dst`, `.search_years` | no |
| `cronnext.instant_of`, `.local_of`, `.is_never`, `.is_dst_affected`, `.dst_note` | no |
| `cronnext.explain_never` | no |
| `crondesc.describe`, `.describe_12_hour`, `.phrases`, `.describe_field` | no |
| `crondesc.cadence`, `.day_rule_note` | no |
| `crondesc.weekday_name`, `.month_name`, `.clock_time`, `.ordinal_word` | no |
| `cronerr.CronError.message`, `cronerr.field_of`, `.offset_of` | no |
| `cronerr.field_name`, `.field_bounds` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
