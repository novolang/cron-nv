# Changelog

All notable changes to cron-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-16

README rewritten to the package README style guide
(docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `cronexpr` — a line parsed once into seven fields and the dialect it
  was read under, with the source kept so a scheduler can log what an
  operator wrote.
- `cronfield` — a column as a sixty-bit set for matching and as the
  terms it was written as for describing, with the Quartz extensions
  as terms because none of them is a fixed set of days.
- `cronnext` — `CronFire`, `CronDst`, and the search in civil time
  with the zone applied at the end.
- `crondesc` — the sentence, and the labelled pieces it is assembled
  from so another language can be built from the same parts.
- `cronerr` — every refusal naming its field, and its column where
  there is one.

### Known

- **`CronFire` is the load-bearing interface.** A cron expression is a
  statement about wall clocks, so a next fire cannot be an interval
  added to the last one, and a fire that daylight saving moved or
  duplicated has to be reported rather than silently resolved.
- **The daylight-saving policy is a value**, and the three are real
  implementations' behaviour rather than a menu.
- **One half of Vixie's rule is named, not approximated**: a wildcard
  hour fires once per wall-clock occurrence, which is two policies at
  once and has no value here.
- **The two day fields are a union when both are restricted.**
  `0 0 13 * 5` is the 13th and every Friday, not Friday the 13th.
- **There is no clock**, and the host row that would run a schedule
  does not exist on the grid yet — the README says what it would be.
- **No device claim**, because a schedule's arithmetic is calendar-nv's
  and calendar-nv makes none.
- **Two `core` dependencies**, calendar-nv by registry range and tz-nv
  by a sibling path while the two are developed together; the path
  becomes `^0.0.1` before the first publish.
