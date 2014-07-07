.. DalmatinerDB benchmarks, created by
   Heinz N. Gies on Sat Jul  7 16:49:03 2014.

Benchmarks
==========

Artifical benchmarks only show a portion of the reality so instead this page shows the result form real workloaded seen in production. The exact same workload is mirrored to different backedns on identical hardare and os configuration.


.. warning::
   
   The benchmarks are first results and not perfected yet, it is by no means a comprehensive compairison but should give a general idea of the state of things.

Setup
-----

Hardware & OS
`````````````

Tests run in Zones on SmartOS 20140124T065835Z, each zone has 16 GB of memory, 400% (non competeing) CPU CAP (and shares equivalent to 4 cores on a hyperthreaded dual `E5-2687W v2 @ 3.40GHz`).

Each zone has 15TB of storage on spinning disks with L2ARC and ZIL on mirrord SSDs.

Configuration
`````````````

DalmatinerDB is configured to use 30 UDP listeners and caches up to 100s in memory before flushing a metric (however most metrics are flushed earlyer a detailed breakdown is given in the section for DalmatinerDB).

Datastores
``````````

* DalmatinerDB - obviously
* Grafite - (based on the SmartOS dataset version)
* KairosDB - (on cassandra 2.0.5)
* InfluxDB - not included, doesn't compile on SmartOS and running in a KVM gives a unfair disadvantage

Workload
--------

The workload the systems are subjected to are a stream of roughly 14000 metrics per seconds (that is one datapoint per metric per second). The only batching allowed is on the metric axis not the time axis, this cuts network overhead and does not reduce the liveliness of the databases data.

Each metric consists of:

* A matric identifyer (name)
* A epoch timestamp
* A value forthat timestamp

Runtime
-------

Since the metric flow is consistant the runtime is considerably long, the mesurements presented here are taken afte a week of continued load.

Results
-------

Results for Grafite are not listed since half way thorugh the test carbon-cache locked up with all memory in the zone consumed.

Usage during opperations
`````````````````````````````


+-------------+--------------+---------------------+
| Measurement | DalmatinerDB | KairosDB            |
+-------------+--------------+---------+-----------+
|             |              |  Kairos | Cassandra |
+-------------+--------------+---------+-----------+
| CPU Usage % | 2.8%         | 3.5%    | 1.8%      |
+-------------+--------------+---------+-----------+
| CPU Cores   | 0.9          | 1.1%    | 0.6       |
+-------------+--------------+---------+-----------+
| Memory SIZE | 260MB        | 1323MB  | 7720MB    |
+-------------+--------------+---------+-----------+
| Memory RSS  | 135MB        | 1303MB  | 5546MB    |
+-------------+--------------+---------+-----------+


Data Size
`````````

All systems use compression shown is the effective datasize on disk:

.. note::

  KairosDB uses compression on Cassandra level and there is no option to disable it, this makes it hard to say waht exactly the compresison ratio is or the effective size per metric is.

+---------------+--------------+-----------+
| Measurement   | DalmatinerDB | KairosDB  |
+---------------+--------------+-----------+
| grows 10m     | 9133B        | 80514B    |
+---------------+--------------+-----------+
| compressratio | 8.32x        | 1.02x *   |
+---------------+--------------+-----------+
| size/point    | 8.65 bit     | ???       |
+---------------+--------------+-----------+


Addendum
--------

DalmatierDB write sizes
```````````````````````
Actual distribution of write cache as affected by read and out of order flushs

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
