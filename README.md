# cron-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`. Installing this package works;
calling it panics with `not implemented`.

## What this is

Crontab expressions as values, and the arithmetic that says when one
fires next. No clock, no daemon, no sleeping: the instant to search
from is an argument, which is what makes a schedule testable without
waiting for it.

- `cronexpr` — a line parsed once into a schedule;
- `cronfield` — one column, as a set and as the terms it was written as;
- `cronnext` — the next and previous fire, in a zone;
- `crondesc` — the schedule in words a person can check;
- `cronerr` — what is wrong, and which field it is wrong in.

```
novo pkg add cron-nv
novo pkg build
novo test
```

## The one example that will work

```novo ignore
use cronerr
use cronexpr
use cronnext
use tzdata

// The next time a schedule fires, and whether the clocks moved it.
fn when_next(expr: Str, zone_name: Str, now: Int) -> Str
    let db = tzdata.data_compact()
    match tzdata.zone(db, zone_name)
        Err(_) => "no such zone"
        Ok(z)  =>
            match cronexpr.parse(expr)
                Err(e) => e.message()
                Ok(s)  =>
                    let fire = cronnext.next_after(s, z, now, cronnext.default_dst())
                    match cronnext.instant_of(fire)
                        None     => "never"
                        Some(at) => str.from_int(at) + cronnext.dst_note(fire)
```

## The load-bearing interface: `CronFire`

```novo ignore
pub enum CronFire
    CronFireAt(utc_second: Int, local: CivilDateTime, offset_seconds: Int)
    CronFireShifted(utc_second: Int, local: CivilDateTime, offset_seconds: Int,
                    wanted: CivilDateTime)
    CronFireRepeat(utc_second: Int, local: CivilDateTime, offset_seconds: Int,
                   occurrence: Int)
    CronFireNever
```

**A cron expression is a statement about wall clocks.** `0 9 * * 1`
means nine in the morning, local, every Monday. The UTC instant that
means moves by an hour twice a year; the wall time does not move at
all. Two things follow, and they are the whole shape of this package.

**A next fire cannot be computed by adding a duration.** `previous +
604800` is right for fifty weeks of the year and an hour out for the
other two, and the error is invisible: the job simply runs at 08:00 for
six months and nobody files anything. So `next_after` searches the
*civil* calendar and applies the zone at the end — which is why it
takes a zone, and why tz-nv is a dependency rather than a convenience.

**And a fire can land on a wall time that did not exist, or on one that
happened twice.** An `Int` can say when; it cannot say *"your 02:30
backup did not run today, because the clocks skipped it"*, which is the
one thing an operator needs to hear. Every variant carries the instant
a scheduler sleeps until, and the three that are not ordinary carry
what happened as well. `dst_note` writes the sentence, once, here,
rather than in every scheduler that consumes this.

`CronFireNever` is the fourth, and it is an answer rather than an
error: `0 0 30 2 *` is a well-formed way to say the 30th of February.
The search is bounded — `search_years()` is four — so "never" means
"not in the next four years", and the bound is public because that
distinction matters to a caller deciding whether to keep the job.

## The daylight-saving policy is a value, and it is real behaviour

| `CronDst` | a wall time that does not exist | a wall time that happens twice |
| --- | --- | --- |
| `CronDstSkip` | does not fire | fires once |
| `CronDstRunOnce` | fires at the instant the clock jumped to | fires once |
| `CronDstRunBoth` | does not fire | fires **twice** |

`CronDstRunOnce` is the default because it is Vixie cron's rule for a
job whose hour and minute are both fixed, and therefore the rule an
operator's expectations were formed by. `CronDstRunBoth` is what a
metering job wants: the hour really did happen twice and there are two
hours of readings in it.

**One half of Vixie's rule is not implemented, and is named rather than
approximated.** A job with a *wildcard* hour — `*/10 * * * *` — runs
once per wall-clock occurrence in cron, so the repeated hour fires its
ten times twice and the skipped hour fires none. That is
`CronDstRunBoth` for the repeat and `CronDstSkip` for the gap at the
same time, and this interface has no value for the combination.

## The two day fields are a union, which is the rule everybody gets wrong

When day-of-month and day-of-week are **both** restricted, cron fires
on **either**. When one is `*`, it constrains nothing:

```text
0 0 13 * 5      midnight on the 13th, AND on every Friday
0 0 13 * *      midnight on the 13th
0 0 * * 5       midnight on every Friday
```

The first is not "Friday the 13th". It is POSIX's rule and Vixie's
implementation, it is what every crontab on every server does, and a
library that quietly ANDed them would produce a schedule that fires a
twelfth as often as the operator expected. `fires_on_either_day` is the
predicate, and `crondesc.day_rule_note` says it out loud in the
sentence a user interface prints.

## Three dialects, chosen and not guessed

| `CronDialect` | fields | extensions |
| --- | --- | --- |
| `CronDialectUnix` | 5 | `@daily` and family; no `L`, `W`, `#`, `?` |
| `CronDialectSixField` | 6 | seconds in front; no extensions |
| `CronDialectQuartz` | 6 or 7 | `L`, `W`, `#`, `?`, and a year field |

The dialect is an argument because `0 0 1 1 *` is a valid five-field
line — midnight on New Year's Day — and a valid six-field one — the
first second of every minute in January — and the two are not close. A
parser that counted the tokens would silently pick one.

The Quartz extensions are flagged rather than merged in:
`uses_extensions` is the question a tool asks before writing a line
into a Unix crontab, because an expression with a `#` in it is a line
that works in this library and silently does nothing on the machine it
is installed on.

## A field is a set, and it keeps its terms

`*/15`, `0,15,30,45` and `0-59/15` are three spellings of the same
sixty-bit set, and once parsed they match identically. The **terms**
survive anyway, for two reasons: `crondesc` says *"every 15 minutes"*
rather than *"at minute 0, 15, 30 and 45"*, and a configuration tool
rewriting one field of a line has to leave the other four looking like
what the operator wrote.

And the Quartz extensions cannot be a set at all, which is the third
reason. `L` is 28, 29, 30 or 31 depending on the month; `5#3` is a
different date every month; `15W` may be the 14th or the 16th.
`has_dynamic_terms` is how a caller finds out that a field's bits are
not the whole answer.

## The host that runs a schedule is a missing row

This package computes; it never runs anything. The half that reads a
clock, sleeps until the next fire, and does something when it arrives
is a `host` package, and **there is no row for it on the grid yet** —
the plan has `cron-nv` in `time`/`core` and nothing above it.

What that row would be: a scheduler over chrono-nv's clock and this
package's arithmetic, with a misfire policy for a process that was
asleep when a fire came due, in `time`/`host`. `tokio-cron-scheduler`
and APScheduler are the reference APIs. Until it exists, a program
holds its own loop and calls `next_after` — which is a perfectly good
arrangement, and the reason this package is useful before that row is
written.

## The layer, and the two dependencies

`core`, and the absence to check for is the clock. There is none:
`next_after` takes the instant to search from as an argument. That is
what keeps the effect row `[]`, and it is also what makes a schedule
testable — every assertion in `tests/` names an instant, and none of
them waits for anything.

calendar-nv is the civil arithmetic — what weekday the 29th is, how
many days February has. tz-nv is the offset that turns a wall time into
an instant. A cron expression needs both, in that order, and the design
of `cronnext` is that the search runs in civil time and the zone is
applied at the end. Both are `core`, so `dep-layer` holds.

**No device claim.** A schedule's arithmetic is calendar-nv's, and
calendar-nv makes no device claim of its own, so a probe naming one of
its types would not link. The honest form of that is no
`tests/embedded_probe.nv` at all rather than a claim narrowed until it
builds.

## The reference implementation

Vixie cron for the five-field dialect and the day-field union rule,
croniter for the next-fire search and its bound, and Quartz for the
extensions. The expressions in `tests/` come from croniter's
`test_croniter.py` and Quartz's `CronExpressionTest`; where the two
disagree, the dialect is what decides, which is the argument for
`CronDialect` being a value.

## Status

**NOT IMPLEMENTED — interface only.** `0.0.1`, `stability = "draft"`,
recorded `implemented = false` on the registry. The first
implementation is the `0.1.0` published over it.

```
novo pkg build     # clean: the signatures type-check and the rows fit
novo test          # red: every body is a todo()
```
