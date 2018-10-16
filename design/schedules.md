# Schedules

We already support `trigger` and (soon) `pr` triggers.
This spec covers scheduled triggers.
By way of introduction, here's a rich example of the syntax:

```yaml
schedules:
- cron: "0 0 * * *"
  displayName: Daily midnight build
  branches:
    include:
    - master
    - releases/*
    exclude:
    - releases/ancient/*
- cron: "0 12 * * 0"
  displayName: Weekly Sunday build
  branches:
    include:
    - releases/*
  batch: true
  always: true
```

These definitions say:
- For `master` and all `releases/*` branches except those under `releases/ancient/*`, run a pipeline at midnight every day.
- For all `releases/*` branches, run a pipeline at noon on Sundays.
  - Only run against branches which have changed (the `always` keyword).
  - Only run one instance of the pipeline per branch (the `batch` keyword). If a branch already has an active run, queue the pipeline instead of running it immediately.

## Supported cron syntax

```
mm HH DD MM DW
 \  \  \  \  \__ day of week
  \  \  \  \____ month
   \  \  \______ day
    \  \________ hour
     \__________ minute
```

[NCrontab's set](https://github.com/atifaziz/NCrontab/wiki/Crontab-Expression) looks like a solid start.

Priority | Type | Example 
----------|------|-------
Must | Wildcard | `*` 
Must | Single values | `5`
Must | Comma-delimited values | `1,2,3,5`
Must | Ranges | `3-6`
Must | Named values | `January`, `Jan`, `Monday`, `Mon`
Nice | Interval specs | `*/4`
Nice | Mixed expressions | `1-10/2,15,31`

Accepted values are:

Field | Range
------|------
Minutes | 0 through 59
Hours | 0 though 23
Days | 1 through 31
Months | 1 through 12, full English names, first three letters of English names
Day of week | 0 through 6, full English names, first three letters of English names

Idea: support negative dates to mean "days from end of month".
For instance, "-2" means 2 days before the end of the month.

## Feature Q & A

### Why cron syntax?
Cron syntax is a concise and clear way to express timing.
For someone unaccustomed to it, there are many tutorials available which quickly get you up to speed.

### What timezone is used?
The Azure DevOps organization has a setting for timezone which we use.
In the future, we may add an explicit `timezone` keyword which would allow for alternate timezones.
