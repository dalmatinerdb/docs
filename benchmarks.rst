.. DalmatinerDB benchmarks, created by
   Heinz N. Gies on Sat Jul  7 16:49:03 2014.

Benchmarks
==========

Artificial benchmarks only show a portion of reality so all benchmarks were made in a real environment and results shown are based on workloads seen in production. To ensure comparability of the results for each database the exact same workload was mirrored to different backends on identical hardware and OS configuration. 


.. warning::
   
   The benchmarks show initial results. The results are by no means a comprehensive comparison but should give a general idea of the state of things.

Setup
-----

Hardware & OS
`````````````

Tests run in Zones on SmartOS 20140124T065835Z, each zone has 16 GB of memory, 400% (non-competing) CPU CAP (and shares equivalent to 4 cores on a hyperthreaded dual `E5-2687W v2 @ 3.40GHz`).

Each zone has 15TB of storage on spinning disks with L2ARC and ZIL on mirrored SSDs.

Configuration
`````````````

DalmatinerDB is configured to use 30 UDP listeners and caches up to 100s in memory before flushing a metric (however most metrics are flushed earlier. A detailed breakdown of this process is given in the DalmatinerDB section) and a R/N/W value of 1.

KairosDB 0.9.3 and Cassandra version 2.0.5 are used with the default configuration (Keyspace configuration based on the KairosDB defaults) with a N value of 1. Since it is not possible to disable compression lz4 is automatically used.

Datastores
``````````

* DalmatinerDB - obviously
* Graphite - (based on the SmartOS dataset version)
* KairosDB - (on Cassandra 2.0.5)
* InfluxDB - not included, doesn't compile on SmartOS and running in a KVM would result in an unfair disadvantage

Workload
--------

The systems are subjected to a workload of a stream of roughly 14,000 metrics per seconds (one datapoint per metric per second). The only batching allowed is on the metric axis not the time axis, which cuts network overhead and does not reduce the liveliness of the database's data.

Each metric consists of:

* A metric identifier (name)
* An epoch timestamp
* A value for the timestamp

Duration
--------

Measurements presented here are taken after a week of continuous load.

Results
-------

Results for Graphite are not listed since halfway through the test carbon-cache locked up with and consumed 100% memory in its zone.

Usage during operations
```````````````````````

The graphs shouw measurements over 1 hour, with the average over a minute taken of a from the described system w/o significant read quaries happening during that timeframe on DalmatinerDB and without any read quaries on KairosDB.

The first graph shows CPU cores used. In it a cpu usage of 3.125 corresponds to 100% of a core used.

+-------------+--------------+---------------------+
| Measurement | DalmatinerDB | KairosDB            |
+-------------+--------------+---------+-----------+
|             |              |  Kairos | Cassandra |
+-------------+--------------+---------+-----------+
| CPU Usage % | 2.8%         | 3.5%    | 1.8%      |
+-------------+--------------+---------+-----------+
| CPU Cores   | 0.9          | 1.1%    | 0.6       |
+-------------+--------------+---------+-----------+

.. image:: _static/img/bench_cpu.png

The second graph shows MB of used memory.

+-------------+--------------+---------------------+
| Measurement | DalmatinerDB | KairosDB            |
+-------------+--------------+---------+-----------+
|             |              |  Kairos | Cassandra |
+-------------+--------------+---------+-----------+
| Memory SIZE | 260MB        | 1323MB  | 7720MB    |
+-------------+--------------+---------+-----------+
| Memory RSS  | 135MB        | 1303MB  | 5546MB    |
+-------------+--------------+---------+-----------+

.. image:: _static/img/bench_mem.png

Data Size
`````````

All systems use compression. Shown is the effective data size on disk:

.. note::

  KairosDB uses compression in Cassandra and there is no option to disable it. This makes it hard to determine the compresison ratio and the effective size per metric.

+---------------+--------------+-----------+
| Measurement   | DalmatinerDB | KairosDB  |
+---------------+--------------+-----------+
| grows 10m     | 9133B        | 80514B    |
+---------------+--------------+-----------+
| compressratio | 8.32x        | 1.02x *   |
+---------------+--------------+-----------+
| size/point    | 8.65 bit     | ???       |
+---------------+--------------+-----------+

Query Times
```````````

Query performed: The maximum nwait per second over the last hour for a given VM.

.. code-block::
   sql

   SELECT max(cloud.zones.cpu.nwait.e2be6f6c-2005-4f2d-aff9-f427b9 BUCKET tachyon, 1m) LAST 1h

+---------------+--------------+-----------+
|               | DalmatinerDB | KairosDB  |
+---------------+--------------+-----------+
| First         | ~2ms         | ~300ms    |
+---------------+--------------+-----------+
| consecutive   | ~1.3ms       | ~135ms    |
+---------------+--------------+-----------+


Query performed: The maximum nwait per hour over the last day for 7 VMs.

.. code-block::
   sql

   select
     max(cloud.zones.cpu.usage.f242021c-c5eb-4c53-a609-64bee4 BUCKET tachyon, 1h),
     max(cloud.zones.cpu.usage.b02df988-2abf-4364-8f55-c39eb3 BUCKET tachyon, 1h),
     max(cloud.zones.cpu.usage.7d1a1a3b-f3e9-4388-a938-c3a866 BUCKET tachyon, 1h),
     max(cloud.zones.cpu.usage.986ea915-f274-41c4-9ac5-b3dbd1 BUCKET tachyon, 1h),
     max(cloud.zones.cpu.usage.1333cf62-b8f1-496a-b2e1-5ec9d4 BUCKET tachyon, 1h),
     max(cloud.zones.cpu.usage.c6a34e43-a242-46e5-89af-b25431 BUCKET tachyon, 1h),
     max(cloud.zones.cpu.usage.e86f77ef-27a3-44c2-9348-f2319b BUCKET tachyon, 1h) LAST 1d

+---------------+--------------+-----------+
|               | DalmatinerDB | KairosDB  |
+---------------+--------------+-----------+
| First         | ~120ms       | ~1600ms   |
+---------------+--------------+-----------+
| consecutive   | ~85ms        | ~1450ms   |
+---------------+--------------+-----------+


Addendum
--------

DalmatierDB write sizes
```````````````````````

Actual distribution of write cache as affected by read and out of order flushes:

=========== ============
# Metrics      # Writes
----------- ------------
38                3
85               10
49               32
16               69
37              132
83              149
84              417
15              588
62              672
93              672
35              682
63              682
69              682
13              806
14              849
36             1030
12             4030
11             4398
9            11694
1            11719
8            12780
10            13124
3            15206
7            25545
6            29203
101          37089
4            52765
5            85455
2            86841
=========== ============
