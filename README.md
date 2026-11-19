# egg-timezone

Time zones for [Egg](https://github.com/femto/egg): the offset from UTC
in force at an instant, from the system's time zone database (TZif files,
RFC 8536), the local zone, a POSIX TZ rule or a fixed offset. Egg's
`std/time` is the calendar in UTC; this package is the zones.

```
clutch add timezone
```

```egg
uses io, alloc, fs, time

const time = @import("std/time")
const zone = @import("timezone")

fn main(io: Io, files: Fs, clock: Clock, a: Allocator) !void {
    const z = try zone.load(files, a, "America/New_York")
    defer a.free(z)
    const now = clock.now()
    var out: Buf(u8) = .empty
    defer a.free(out)
    time.append_rfc3339(a, &out, now, z.offset(now))
    io.print(out, " ", z.abbr(now))     // 2026-11-19T01:45:57.444639-05:00 EST
}
```

It needs [Egg](https://github.com/femto/egg) 0.2.2 or later, for
`std/time`, the calendar it builds on. In a project's `clutch.toml`:

```toml
[dependencies]
timezone = "0.1"
```

## Zones

A `zone.Zone` owns what it read; `a.free(z)` frees it.

| | gives |
| --- | --- |
| `zone.load(fs, a, name)` | the zone `name` ("America/New_York") from the system's TZif files; `zone.utc()` for "UTC"; `error.UnknownZone` when no file has it or the name could leave the database ("../x", "/etc/x"), `error.BadZone` when its file is not TZif |
| `zone.local(fs, args, a)` | the local zone: `TZ` (a zone name, `:name`, or a POSIX rule), else `/etc/localtime`; anything it cannot read is UTC, as in Go |
| `zone.posix(a, rule)` | a POSIX TZ rule, such as `"EST5EDT,M3.2.0,M11.1.0"` |
| `zone.fixed(a, name, offset)` | a fixed offset, seconds east of UTC, from -24:59:59 to +25:59:59 (RFC 8536's range); `error.BadZone` outside it |
| `zone.parse_tzif(a, data)` | TZif data, version 1 to 4 |
| `zone.utc()` | UTC |

| `z.` | gives |
| --- | --- |
| `offset(ns)` | the seconds east of UTC in force at the instant `ns`, for `time.append_rfc3339` |
| `abbr(ns)` | its abbreviation ("EST") |
| `dst(ns)` | whether it is daylight saving time |
| `at(ns)` | the `time.DateTime` there |
| `instant(t)` | `?i64`: the instant of the local time `t`; a time skipped when clocks go forward takes the offset from before, one repeated when they go back is the first (Python's `fold=0`) |

Windows has no zone files, so there `zone.load` knows only UTC and
`zone.local` is UTC unless `TZ` holds a POSIX rule.

## Tests

```
clutch test
```

`test/zone.egg` prints what `test/zone.expected` holds. Its TZif data and
expected offsets were checked against Python's zoneinfo.
