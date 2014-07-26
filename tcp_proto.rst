.. DalmatinerDB data input manual
   Heinz N. Gies on Sat June 5 16:49:03 2014.

TCP Protocol
============

Keepalife
---------

A simple keepalife that can be send, no reply will be send to this message

.. code-block:: erlang

   <<0>>.

List Buckets
------------

This command list all buckets, each bucket known to the system. The command is received and a reply send directly.

.. code-block:: erlang

   <<3>>.

The Reply is prefixed with the total size of the whole reply in bytes (not including the size prefix itself). Then each bucket is prefixed by a size of the bucket name.

.. code-block:: erlang

   %% Outer wrapper
   <<ReplySize:32/integer, Reply:ReplySize/binary>>.
   %% Elements of the reply
   <<BucketSize:16/integer, Bucket:BucketSize/binary>>.


List Metrics
------------

Lists all metrics in a bucket. The bucket to look for is prefixed by 1 byte size for the bucket name.
.. code-block:: erlang

   <<1,
     BucketSize:8/integer, Bucket:BucketSize/binary>>.

The Reply is prefixed with the total size of the whole reply in bytes (not including the size prefix itself). Then each metric is prefixed by a size of the metric name.

.. code-block:: erlang

   %% Outer wrapper
   <<ReplySize:32/integer, Reply:ReplySize/binary>>.
   %% Elements of the reply
   <<MetricSize:16/integer, Metric:MetricSize/binary>>.


Get
---

Retrieves data for a metric, bucket and metric are size prefixed as strings, Time and count are unsigned integers.

.. code-block:: erlang

   <<2,
     BucketSize:8/integer, Bucket:BucketSize/binary,
     MetricSize:16/integer, Metric:MetricSize/binary,
     Time:64/integer, Count:32/integer>>.

There will **always** be returned ``Count`` messages will be returned, if there is no or insufficient data or the bucket/metric doesn't exist the missing data will be filled with blanks.

.. code-block:: erlang

   <<Reply:((9*8)*Count)/signed-integer>>.
