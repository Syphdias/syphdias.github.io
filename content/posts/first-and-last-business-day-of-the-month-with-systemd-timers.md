---
title: "First and Last Business Day of the Month With Systemd Timers"
date: 2023-03-14T00:00:48+01:00
---
I recently was looking for a way to run a systemd service on the last business
day of the month, but I could only find an answer for first business day of
every month on Stack Overflow which was wrong. So I looked into it.

Spoiler: This is not possible with one calendar expression.

If you remember only one thing from this blog post, **remember
`systemd-analyse`**. There are quite a few useful subcommands, e.g. `verify`.
You should check them out, if you do not know them yet with `systemd-analyse
-h`.

For timers, we want `systemd-analyse calendar`.

## First Business Day of the Month
To achieve this we can set `OnCalendar` twice in the timer unit, we do not need
two timer units:
```
[Timer]
OnCalendar=Mon..Fri *-*-01
OnCalendar=Mon *-*-02..03
```

- `Mon..Fri *-*-01` will activate if the first day of the month is a business
  day
- `Mon *-*-02..03` will activate if the second or third day of the month is a
  Monday. This happens if the first was a Sunday, or if the first was a Saturday
  and the second was a Sunday. There is no overlap.

To verify this is true we can use (`--iterations` is optional but very useful):
```sh
systemd-analyze calendar 'Mon..Fri *-*-01' 'Mon *-*-02..03' --iterations 10
```

The output is a bit unwieldy since it treats both expressions separately and
prints three versions of the iterations.
```sh
systemd-analyze calendar 'Mon..Fri *-*-01' 'Mon *-*-02..03' --iterations 10 \
    |grep Iteration |sort -k4
```
This works for a English locale but it will filter out the first iteration
because it is listed as "Next elapse". The get a good overview in this case it
suffices.


## Last Business Day of the Month
Last day is very similar to first day, but requires dates that were introduced
in [systemd v233]. Again, we need two expressions:
```
[Timer]
OnCalendar=Mon..Fri *-*~01
OnCalendar=Fri *-*~02..03
```

- `Mon..Fri *-*~01` triggers if the last day of the month is a business day
- `Fri *-*~02..03` triggers if the second to last or third to last day of the
  month was a Friday. This happens if the weekend is on the last day of the
  month.

Verifying this works just the same:
```sh
systemd-analyze calendar 'Mon..Fri *-*~01' 'Fri *-*~02..03' --iterations 10
```

Optionally, filter and sort with the caveats from above:
```sh
systemd-analyze calendar 'Mon..Fri *-*~01' 'Fri *-*~02..03' --iterations 10 \
    |grep Iteration |sort -k4
```


[systemd v233]: https://github.com/systemd/systemd/blob/v233/NEWS#L174
