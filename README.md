# JSON-TimeSeries: A Format for Time Series Data

Version 0.1 - Working Draft

This informal specification lays out a way to represent time series information
using JSON. The objective is to enable cross-platform sharing of a wide variety
of regular and irregular time series data.

The format puts no hard limits on observation frequency, but library support may
vary.

<!-- TOC depthFrom:2 -->

- [Representing Dates](#representing-dates)
- [Irregular Time Series](#irregular-time-series)
  - [Examples](#examples)
- [Regular Time Series](#regular-time-series)
  - [Base Period Boundary Calculation](#base-period-boundary-calculation)
  - [Examples](#examples-1)
- [Implementations](#implementations)

<!-- /TOC -->

## Representing Dates

Dates are represented using strings. The format for date strings is derived from
ISO 8601, subject to a few minor modifications.

As with ISO 8601, dates are represented in the following format:

`[Year]-[Month]-[Day]T[Hour]:[Minute]:[Second].[SubSecond][TimeZone]`

- `[Year]` is the four-digit year (0000-9999)

- `[Month]` is the two-digit month (01-12)

- `[Day]` is the two-digit day of the month (01-31)

- `[Hour]` is the two-digit hour in 24-hour format (00-23)

- `[Minute]` is the two-digit minute (00-59)

- `[Second]` is the two-digit second (00-59)

- `[SubSecond]` is expressed in milliseconds (000-999), microseconds
  (000000-999999), or to an arbitrary degree of precision with digits added in
  groups of three

- `[TimeZone]` is one of the following:

  - `Z` - indicates UTC

  - `+HH:MM` - indicates a positive offset from UTC

  - `-HH:MM` - indicates a negative offset from UTC

For example, to millisecond precision:

`2000-01-01T00:00:00.000Z`

As with ISO 8601, higher-precision date elements may be omitted. Unlike ISO
8601, a stand-alone year of the form `YYYY` is considered a valid date.

For the purpose of this format, dates should be understood to refer to specific
instants in time, not intervals of time, irrespective of the precision inherent
in the date expression.

Lower-precision dates should be understood as follows:

- `YYYY` should be understood as `YYYY-01-01T00:00:00.000`

- `YYYY-MM` should be understood as `YYYY-MM-01T00:00:00.000`

- `YYYY-MM-DD` should be understood as `YYYY-MM-DDT00:00:00.000`

- `YYYY-MM-DDTHH` should be understood as `YYYY-MM-DDTHH:00:00.000`

- `YYYY-MM-DDTHH:NN` should be understood as `YYYY-MM-DDTHH:NN:00.000`

- `YYYY-MM-DDTHH:NN:SS` should be understood as `YYYY-MM-DDTHH:NN:SS.000`

The date `2019` should therefore be understood to represent midnight on January
1st, not the year 2019 as a whole.

As with ISO 8601, the time zone may be omitted, in which case the date is
understood to refer to local time.

Unlike ISO 8601, a time zone specification is permitted even if there are no
time elements in the date string. For example, `2000Z`, `2000-01Z`, and
`2000-01-01Z` all represent midnight on January 1, 2000, UTC.

Other ISO 8601 representations like `YYYY-WXX` and `YYYY-DDD` are not permitted.

## Irregular Time Series

Irregular time series are represented using JSON objects of the following form:

```Javascript
{
  "JsonTs":       "irregular",
  "Observations": ...
}
```

1. The `JsonTs` parameter (required string) is `"irregular"` to indicate the
   nature of the time series. The value is not case-sensitive.

2. The `Observations` parameter (required array) contains the time series
   observations in chronological order. It may be empty, which indicates that
   there are no obsevations.

   Each element of `Observations` must take one of two forms:

   1. `[Start, Value]`

      Representing an observation in this form indicates that the `Start` of the
      following observation should serve as the `End` of the current one. The
      final observation must not take this form.

      - `Start` (date string) specifies the beginning of the time interval to
        which `Value` applies. The instant represented by `Start` is considered
        part of the observation, while the instant represented by the `Start` of
        the following observation is considered part of the next observation.

        For the first observation in a series, any `Start` is valid.

        If following an observation of type (1), `Start` must be strictly later
        than the `Start` of the preceding observation.

        If following an observation of type (2), `Start` must be later than or
        equal to the `End` of the preceding observation.

      - `Value` (any JSON value) specifies the value of the observation.

   2. `[Start, Value, End]`

      Representing an observation in this form allows for an explicit `End`
      date, which can be used to represent gaps in the data. The final
      observation must take this form.

      - `Start` (date string) specifies the beginning of the time interval to
        which `Value` applies. The instant represented by `Start` is considered
        part of the observation.

        For the first observation in the series, any `Start` is valid.

        If following an observation of type (1), `Start` must be strictly later
        than the `Start` of the preceding observation.

        If following an observation of type (2), `Start` must be later than or
        equal to the `End` of the preceding observation.

      - `Value` (any JSON value) specifies the value of the observation.

      - `End` (date string) specifies the end of the time interval to which the
        `Value` applies. The instant represented by `End` is not considered part
        of the observation. `End` must be strictly later than `Start`.

### Examples

1. An irregular time series containing string values from midnight UTC on
   January 1, 2000 until midnight UTC on January 10, 2000, with no gaps in the
   data.

   ```Javascript
   {
     "JsonTs": "irregular",
     "Observations": [
       ["2000Z", "value1"],
       ["2000-01-03T04:00:10Z", "value2"],
       ["2000-01-08T23:40:20Z", "value3", "2000-01-10Z"]
     ]
   }
   ```

2. An irregular time series containing string values from midnight UTC on
   January 1, 2000 until midnight UTC on January 10, 2000, with a gap in the
   data.

   ```Javascript
   {
     "JsonTs": "irregular",
     "Observations": [
       ["2000Z", "value1"],
       ["2000-01-03T04:00:10Z", "value2", "2000-01-04T07:15:30Z"],
       ["2000-01-08T23:40:20Z", "value3", "2000-01-10Z"]
     ]
   }
   ```

## Regular Time Series

Regular time series can in principle be represented using the irregular format,
but that would discard information about the repetitive structure of the time
series, which is often relevant when interpreting and modifying the data. The
regular time series format is designed to retain this information.

Regular time series are represented by JSON objects of the following form:

```Javascript
{
  "JsonTs":       "regular",
  "BasePeriod":   ...,
  "Anchor":       ...,
  "SubPeriods":   ...,
  "Observations": ...
}
```

1. The `JsonTs` parameter (required string) is `"regular"` to indicate the
   nature of the time series. The value is not case-sensitive.

2. The `BasePeriod` parameter (required array) specifies the base periodicity of
   the series. Base periods can be broken into higher frequency observations
   using the `SubPeriods` parameter.

   The `BasePeriod` parameter must take the following form...

   `[BasePeriodNumber, BasePeriodType]`

   Where...

   - `BasePeriodNumber` (positive integer) specifies the number of
     `BasePeriodType`s that constitute a base period.

   - `BasePeriodType` (string) must be one of:

     - `y` - Years
     - `m` - Months
     - `w` - Weeks
     - `d` - Days
     - `h` - Hours
     - `n` - Minutes
     - `s` - Seconds
     - `ms` - Milliseconds
     - `e-#` - Arbitrarily high frequency

     The `e-#` base period type enables the representation of time series with
     arbitrarily high frequency. `#` must be a positive multiple of 3. The
     expression uses scientific notation to indicate the length of the base
     period in terms of seconds, so `e-3` is equivalent to `ms` and `e-6`
     indicates microsecond precision. There is no format-prescribed limit on
     `#`, but library support varies.

     The `BasePeriodType` is not case-sensitive.

   Examples:

   - `[1, "Y"]` indicates a base period of one year
   - `[5, "n"]` indicates a base period of five minutes

3. The `Anchor` parameter (optional date string) specifies the starting instant
   of one base period and, indirectly, the starting instants of all other base
   periods. The `Anchor` parameter allows an entire series to be shifted forward
   or backward in time.

   For example, if `BasePeriod` is `[1, "y"]` and `Anchor` is `"2000-01Z"`, then
   each base period will run from the start of January 1st UTC through the end
   of December 31st UTC. If `BasePeriod` is `[1, "y"]` and `Anchor` is
   `"2000-07Z"` then each base period will run from the start of July 1st UTC
   through the end of June 30th UTC of the next year.

   If the `Anchor` parameter is not specified...

   - If `BasePeriodType` is `"w"`, then `Anchor` must default to midnight UTC on
     a Monday. For example, `"2000-01-03T00:00:00Z"`.

   - If `BasePeriodType` is anything else, then `Anchor` must default to
     midnight UTC on a January 1st. For example, `"2000-01-01T00:00:00Z"`.

4. The `SubPeriods` parameter (optional positive integer) specifies the number
   of observations within each `BasePeriod`.

   For example, a time series with business-day frequency would have
   `BasePeriod` set to `[1, "w"]` and `SubPeriods` set to `5`.  
    If the `SubPeriods` parameter is not specified, it must default to `1`.

5. The `Observations` parameter (required array) contains the time series
   observations in chronological order. It may be empty, which indicates that
   there are no obsevations.

   Each element of `Observations` must take one of three forms:

   1. `[BasePeriodDate, SubPeriodNumber, Value]`

      Representing an observation in this form allows its base period and sub
      period to be specified explicitly.

      `BasePeriodDate` (date string) indicates the base period to which the
      observation is being assigned. It may reference any instant falling within
      the base period, as determined by the boundary calculation procedure
      below.

      `SubPeriodNumber` (positive integer) specifies the sub period to which
      `Value` is being assigned. Sub periods are indexed beginning at `1`, so
      `SubPeriodNumber` must fall between `1` and `SubPeriods`.

      `Value` (any JSON value) is the value of the observation.

      Observations represented in this form must be ordered chronologically...

      - If adjacent observations reference different base periods, the base
        period referenced by the second observation must be later than the base
        period referenced by the first.

      - If adjacent observations reference the same base period, the second
        observation must have a strictly greater `SubPeriodNumber`.

   2. `[BasePeriodDate, Value]`

      Representing an observation in this form is equivalent to
      `[BasePeriodDate, 1, Value]`. Observations may only be represented in this
      form if `SubPeriods` is 1.

   3. `[Value]`

      Representing an observation in this form indicates that it occupies the
      sub period immediately following the previous observation. If the previous
      observation had a `SubPeriodNumber` equal to `SubPeriods`, the observation
      will occupy the first sub period of the next base period.

      The first observation must not take this form.

      `Value` (any JSON value) is the value of the observation.

### Base Period Boundary Calculation

Base period boundaries are determined by adding and subtracting the interval
specified in `BasePeriod` to the `Anchor`, which by definition is the beginning
of a base period.

To calculate the boundaries of base periods after the anchor:

- The first base period after the anchor starts at `Anchor` and ends immediately
  prior to `Anchor + BasePeriod`.

- The second base period after the anchor starts at `Anchor + BasePeriod` and
  ends immediately prior to `Anchor + 2 * BasePeriod`.

- And so on...

To calculate the boundaries of base periods before the anchor:

- The first base period before the anchor starts at `Anchor - BasePeriod` and
  ends immediately prior to `Anchor`.

- The second base period before the anchor starts at `Anchor - 2 * BasePeriod`
  and ends immediately prior to `Anchor - BasePeriod`.

- And so on...

When adding to or subtracting from a date element on the anchor, less-granular
date elements should be rolled forward or backward as appropriate and
more-granular date elements should be left unchanged. When the anchor lies on
the 29th, 30th, or 31st of the month and the month resulting from a boundary
calculation does not contain that day, the calculation should substitute the
last day of that month.

Sub periods are indexed ordinally within each base period, but the time
boundaries of each sub period within its base period are not prescribed by the
format. For maximum flexibility, this is left to the application/library.

### Examples

1. A monthly time series containing numerical values for the first three months
   of the year 2000.

   ```Javascript
   {
     "JsonTs": "regular",
     "BasePeriod": [1, "m"],
     "Observations": [
       ["2000-01", 1],
       [2],
       [3]
     ]
   }
   ```

2. A time series with ten-minute frequency containing string values for the
   first and last twenty minutes of 2019

   ```Javascript
   {
     "JsonTs": "regular",
     "BasePeriod": [10, "n"],
     "Observations": [
       ["2019-01-01T00:00:00Z", "A"],
       ["B"],
       ["2019-12-31T23:40:00Z", "Y"],
       ["Z"]
     ]
   }
   ```

3. A time series containing numerical fiscal quarter observations for one fiscal
   year ending October 31.

   ```Javascript
   {
     "JsonTs": "regular",
     "BasePeriod": [1, "q"],
     "Anchor": "2000-11-01",
     "Observations": [
       ["2000-11-01", 100],
       [200],
       [300],
       [400]
     ]
   }
   ```

4. A weekly time series with five weeks of boolean observations, where weeks
   start on Sunday and end on Saturday.

   ```Javascript
   {
     "JsonTs": "regular",
     "BasePeriod": [1, "w"],
     "Anchor": "2019-01-06",
     "Observations": [
       ["2019-01-06", 1, true],
       [false],
       [true],
       [false],
       [true]
     ]
   }
   ```

5. A business week time series with one week of numerical data.

   ```Javascript
   {
     "JsonTs": "regular",
     "BasePeriod": [1, "w"],
     "SubPeriods": 5,
     "Observations": [
       ["2000-01-03", 1, 1],
       [2],
       [3],
       [4],
       [5]
     ]
   }
   ```

6. A business week time series with data for Monday, Tuesday, Thursday, and
   Friday. Note that the observation for Thursday references Monday, January 3rd
   as the base period, but could equivalently reference Thursday, January 6th,
   as both fall within the same weekly base period.

   ```Javascript
   {
     "JsonTs": "regular",
     "BasePeriod": [1, "w"],
     "SubPeriods": 5,
     "Observations": [
       ["2000-01-03", 1, 1],
       [2],
       ["2000-01-03", 4, 4],
       [5]
     ]
   }
   ```

7. A time series with millisecond frequency containing two string observations
   in the middle of the first second of 2000.

   ```Javascript
   {
     "JsonTs": "regular",
     "BasePeriod": [1, "e-3"],
     "Observations": [
       ["2019-01-01T00:00:00.499Z", "first"],
       ["second"]
     ]
   }
   ```

## Implementations

[Javascript: Chronology.js](https://github.com/aarong/chronology)
