.. DalmatinerDB Query Language
   Heinz N. Gies on Sat Jul  5 16:49:03 2014.

Dalmatiner Query Language
=========================

General
-------

The front end query language is rather remotely related to SQL, which should make it simple to pick up. The focus is on querying one metric at a time, given that those queries are incredible fast. Globs can also be used to query multiple metrics at a time.

The basic syntax looks like this::

   SELECT <FIELDS> [ FROM <ALIASES>] <TIME RANGE> [IN <RESOLUTION>]

Please keep in mind that each query can only return a single row of data at the moment.

Fields (`SELECT` section)
`````````````````````````

A field can either be a metric, an alias for a metric or a function:

* `cloud.zones.cpu.usage.eca485cf-bdbb-4ae5-aba9-dce767 BUCKET tachyon` - a fully qualified metric.
* `vm` - an alias that is defined in the `FROM` section of the query.
* `avg(vm, 1m)` - a aggregation function.

Fields can be aliased for output adding a `AS <alias>` directive after the field.

Multiple fields can be given separating two fields with a `,`. The resolution of fields does not have to be the same, and there is no validation to enforce this!

Aliases (`FROM` section)
````````````````````````

When a metric is used multiple times it is more readable to alias this metric in the `FROM` section. Multiple elements can be given separated with a `,`. Each element takes the form: `<metric> BUCKET <bucket> AS <alias>`.

It is possible to match multiple metrics by using a mulitget aggregator and a glob to match a metric. Valid multiget aggregators are `sum` and `avg`. For example: `sum(some.metric.* BUCKET b)`.

Time Range
``````````

There are two ways to declare ranges. Although numbers here represent seconds, DalmatinerDB does not care about the time unit at all::

  BETWEEN <start:reltime> AND <end:reltime>


The above statement selects all points between the `start` and the `end`.

The most used query is 'what happened in the past X seconds?', so there is a simplified form for this::

  LAST <amount:int>|[time:time>]

Resolution (`IN` section)
`````````````````````````
By default queries treat incoming data as a one second resolution, however this can be adjusted by passing a resolution section to the query. The syntax is: `IN <resolution:time>`.

Data Types
----------

Integer (int)
`````````````

A simple number literal i.e. `42`.

Time (time)
```````````

There are two ways to declare times:

* Relatively, in which case the time is a simple integer and corresponds to a number of metric points used. (i.e. `60`)
* Absolute, in which case the time is an integer followed by a time unit such as `ms`, `s`, `m`, `h`, `d` and `w`. In this case the resolution of the metric is taken into account.

Relative Time (reltime)
```````````````````````

Relative times can either be the keyword `NOW`, an absolute timestamp (a simple integer) or a relative time in the past such as `<time> AGO`.

Metric
``````

Metrics are simple strings that are optionally separated by dots and a second string for the bucket. The two strings are separated by the keyword `BUCKET`.

Example::

  cloud.zones.cpu.usage.eca485cf-bdbb-4ae5-aba9-dce767 BUCKET tachyon

Aggregation Functions
---------------------

Aggregation functions aggregate a metric over a given range of time and decrease the resolution by doing so. Aggregation functions can be nested, in which case the 'higher' functions work with the decreased resolution of lower functions and not the raw resolution. This means the correct code to get the 1m average over 10s sums from a 1s resolution metric would be  `avg(sum(m, 10s), 1m)` not `avg(sum(m, 10s), 6s)` - however this does not apply when using the point and not the time declaration, so it would be: `avg(sum(m, 10s), 6)` not `avg(sum(m, 10s), 60)` (please note the missing `s`).

min/2
`````
The minimal value over a given range of time.

max/2
`````
The maximal value over a given range of time.

sum/2
`````
The sum of all values of a time-range.

avg/2
`````
The average of a time-range (this is the mean not the median).

empty/2
```````
Returns the total of empty data-points in a time-range. This can be used to indicate the precision of the data and the loss occurring before they get stored.

percentile/3
````````````
Returns the value of the ``n`` th percentile, where 0 < ``n`` < 1. The percentile is given as the second value of the function, the time-range to aggregate over as the third.

Manipulation Functions
----------------------

Manipulation functions help to change the values of a value list they do not change the resolution or aggregate multiple values into one.

derivate/1
``````````
Calculates the derivate of a metric, meaning N'(X)=N(X) - N(X-1)

.. note::
   Even if the resolution isn't changed this function removes exactly 1 element from the result

multiply/2
``````````
Multiplies each element with integer constant.

divide/2
````````
Divides each element with a integer constant.


Examples
--------

Calculates the min, max and average of a metric over a hour:

.. code-block:: sql

   SELECT min(vm, 10m), avg(vm, 10m), max(vm, 10m) AS max FROM cloud.zones.cpu.usage.eca485cf-bdbb-4ae5-aba9-dce767 BUCKET tachyon AS vm LAST 60m
