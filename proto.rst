.. DalmatinerDB data input manual
   Heinz N. Gies on Sat June 5 16:49:03 2014.


Network Protocol
****************

General
=======

Constants / Macros
------------------
The following constants (Macros) are defined as part of the protocol as they name frequently reccuring values:

.. code-block:: erlang

   %% dproto.hrl

   %% number of bits used to encode the bucket size.
   %% => buckets can be 255 byte at most!
   -define(BUCKET_SS, 8).

   %% The number of bits used to encode the size of the a single metric part.
   %% => each part can be 255 byte at max!
   -define(METRIC_ELEMENT_SS, 8).

   %% The number of bits used to encode the size of a whole metric (All of its
   %% parts)
   %% => the maximum number of bytes in a whole metric can be 65,536, so a single
   %% metric can hold at least 255 elments, more if their size is < 256 byte.
   -define(METRIC_SS, 16).

   %% The number of bits used to encode a list of metrics.
   %% => this means a list operation can return at least 281,474,976,710,656
   %% metrics.
   %%
   %% That is a lot! Good problem to have if we ever face it!
   -define(METRICS_SS, 64).

   %% The number of bits used to encode the length of th payload data.
   %% => this means we can encode 4,294,967,296 byte or 536,870,912 points
   %% at 8 byte / point in a single request.
   %%
   %% we should never do that!
   -define(DATA_SS, 32).

   %% Tye number of bits used for encoding the time.
   -define(TIME_SIZE, 64).

   %% The number of bits used for encoding the count.
   -define(COUNT_SIZE, 32).

   %% Number of bits used to encode the delay as part of the streaming protocol.
   -define(DELAY_SIZE, 8).

   %% The type used to encode sizes.
   -define(SIZE_TYPE, unsigned-integer).

   %% The type used to encode time.
   -define(TIME_TYPE, unsigned-integer).

   %% mmath.hrl
   -define(BITS, 56).
   -define(INT_TYPE, signed-integer).

   -define(NONE, 0).
   -define(INT, 1).
   -define(TYPE_SIZE, 8).

   -define(DATA_SIZE, ((?BITS + ?TYPE_SIZE) div 8)).

Datapoints
----------

On both read and write datapoints are encoded as follows:

.. code-block:: erlang

   <<?INT:?TYPE_SIZE, Value:?BITS/?INT_TYPE>>.
   <<?NONE:?TYPE_SIZE, 0:?BITS/?INT_TYPE>>.

.. note::

   Not every language handles 56 bit integers as well as erlang does, however 32 bit integers can be used when padded from the left with three bytes (24 bit) of zeros (``0``) for positive values, or minus one (``-1`` / ``255``) bytes for negative integers. An alternative is using 64 bit integers and discarding the left most byte.

   .. code-block:: erlang

      <<10:56/signed-integer>> = <<0:24/signed-integer, 10:32/signed-integer>>.
      <<0,0,0,0,0,0,10>>
      <<-10:56/signed-integer>> = <<-1:24/signed-integer, -10:32/signed-integer>>.
      <<"ÿÿÿÿÿÿö">>
      <<-1,-1,-1,-1,-1,-1,-10>>

Metric Names
------------

Metric names are not simple strings but a length prefixed list of elements. The upside of this is that there are no reserved characters (such as ``.``) and it allows faster parsing and matching against them.

An example would be:

.. code-block:: erlang

   <<2, "my", 3, "key">>.
   <<3, "yet", 7, "another", 3, "one">>.


Ingress (Stream Mode)
=====================

The TCP endpoint can only accept incoming data when switched to stream mode. This way a connection is dedicated to send data to a single bucket. Flushing can be handled either manually or automatically. Automatic flushing sets a maximum delta between the first data cached for the connection and the newest arrived bit of information.


It is possible to switch the TCP connection, this stream allows to specify a bucket for the stream
and by that prevent it to be resent with every metric. Also, it makes it possible for the connection
cache to have a specified maximal duration between the first and the last metric received before the
data is flushed.

Initializing
------------

This will switch the TCP connection to stream mode. From then on, only payload and flush messages
are accepted.

.. warning::

   Once initialized there is no more 4 byte prefix! This allows for a more efficient way of streaming
   data since even partially arived packages can be handled in a way.


.. code-block:: erlang

   % Identifies entering stream mode.
   <<4,
   % We will flush when the delay is greater or equal Delay
     Delay:?DELAY_SIZE/?SIZE_TYPE,
   % All metrics on this stream will be stored in this bucket.
     BucketSize:?BUCKET_SS/?SIZE_TYPE, Bucket/binary
   >>.

In addition, it is possible to specify a resolution for a bucket when switching
a connection to stream mode:

.. code-block:: erlang

   % Identifies entering stream mode.
   <<4,
   % We will flush when the delay is greater or equal Delay
     Delay:?DELAY_SIZE/?SIZE_TYPE,
   % DataPoints are measured every N number of seconds
     Resolution:?TIME_SIZE/?TIME_TYPE,
   % All metrics on this stream will be stored in this bucket.
     BucketSize:?BUCKET_SS/?SIZE_TYPE, Bucket/binary
   >>.

If a resolution is not supplied, the default value of 1s (1000) will be used.
Since the resolution for a bucket may not be changed once set, the supplied
value cannot differ to that already set for existing buckets.

Payload
-------

The metric packages automatically flush the connection cache when ``(Time - min(All Times)) > MaxDelay``.

The data can hold one or more metric values and it is possible to include 'unset'.

.. code-block:: erlang

   <<5,                                 %% Identifies this as a metric package
     Time:?TIME_SIZE/?SIZE_TYPE,        %% The time offset
     _MetricSize:?METRIC_SS/?SIZE_TYPE, %% Length of the metric name in bytes.
     Metric:_MetricSize/binary,         %% The metric.
     _DataSize:?DATA_SS/?SIZE_TYPE,     %% Length of the data in bytes.
     Data:_DataSize/binary              %% One or more metric points
   >>.

Flush
-----

It is possible to control the flush time outside of the timing by forcing a flush as part of the stream. To do that the ``flush`` message can be used.

.. code-block:: erlang

   <<6>>. % Indicates that at this point the connection cache should be flushed.

Batching
--------
It is possible to batch multiple inserts that are targeted at the same time, this allows to save some extra bandwith when transmitting data. The batching is only available in stream mode.

The batch is initialized with the following message:

.. code-block:: erlang

   <<10,                                %% Command code for batch start
     Time:?TIME_SIZE/?SIZE_TYPE,        %% The time offset
   >>.

This can be followed by as many batch packages are desired, each package include one metric name and a single datapoint:

.. code-block:: erlang

   <<_MetricSize:?METRIC_SS/?SIZE_TYPE, %% Length of the metric name in bytes.
     Metric:_MetricSize/binary,         %% The metric.
     Point:8/binary                     %% One or more metric points
   >>.

When no more datapoints are desired for this batch, the batch can be terminated by sending a 2 0 byte (which would not be a valid payload package since the MetricSize must be at least 1).

.. code-block:: erlang

   <<0:?METRIC_SS/?SIZE_TYPE>>.         %% This would not be a valid payload package.


Querying
========

List Buckets
------------

This command list all buckets known to the system. The command is received and a reply send directly.

.. code-block:: erlang

   <<3>>.

The reply is prefixed with the total size of the whole reply in bytes (not including the size prefix itself). Then each bucket is prefixed by a size of the bucket name.

.. code-block:: erlang

   %% Outer wrapper
   <<ReplySize:?BUCKETS_SS/?SIZE_TYPE, Reply:ReplySize/binary>>.
   %% Elements of the reply
   <<BucketSize:?BUCKET_SS/?SIZE_TYPE, Bucket:BucketSize/binary>>.


List Metrics
------------

Lists all metrics in a bucket. The bucket to look for is prefixed by 1 byte size for the bucket name.

.. code-block:: erlang

   <<1,
   %% The size and the bucket binary to read the metric list from
     BucketSize:?BUCKET_SS/?SIZE_TYPE,
     Bucket:BucketSize/binary
   >>.

The reply is prefixed with the total size of the whole reply in bytes (not including the size prefix itself). Then each metric is prefixed by a size of the metric name.

.. code-block:: erlang

   %% Outer wrapper
   <<ReplySize:32/integer, Reply:ReplySize/binary>>.
   %% Elements of the reply
   <<MetricSize:16/integer, Metric:MetricSize/binary>>.


Reading Data
------------

Retrieves data for a metric, bucket and metric are size prefixed as strings, Time and count are unsigned integers.

.. code-block:: erlang

   <<2,
   %% The Size of the bucket binary and the bucket itself
     BucketSize:?BUCKET_SS/?SIZE_TYPE,
     Bucket:BucketSize/binary,
   %% The Size of the metric binary and the bucket itself
     MetricSize:?BUCKET_SS/?SIZE_TYPE,
     Metric:MetricSize/binary,
   %% The start time to read from (given in bucket resolution)
     Time:?TIME_SIZE/?SIZE_TYPE,
   %% The number of points to read.
     Count:?COUNT_SIZE/?SIZE_TYPE
   >>.

There will **always** be returned ``Count`` messages will be returned, if there is no data or insufficient data or the bucket/metric doesn't exist the missing data will be filled with blanks.

.. code-block:: erlang

   <<Reply:(?DATA_SIZE*Count)/binary>>.


where each of the elements looks like one of these:


Bucket Information
------------------

Gets information of the bucket, namely the resolution and the points per file.

.. warning::

   Not yet implemented.

.. code-block:: erlang

   <<7,
   %% The Size of the bucket binary and the bucket itself
     BucketSize:?BUCKET_SS/?SIZE_TYPE, Bucket:BucketSize/binary
   >>.

The reply will return the resolution and the points per file of the bucket.

.. code-block:: erlang

   <<
     Resolution:?TIME_SIZE/?TIME_TYPE, %% The resolution of the bucket
     PPF:?TIME_SIZE/?TIME_TYPE         %% The points per file of the bucket
     TTL:?TIME_SIZE/?TIME_TYPE         %% The time a bucket will retain data, 0 indicates indefinite
   >>.

Management
==========

Adding a bucket
---------------

Adding a bucket can be achived by the following call.

.. warning::

   Not yet implemented.

.. code-block:: erlang

   <<8,
   %% The Size of the bucket binary and the bucket itself
     BucketSize:?BUCKET_SS/?SIZE_TYPE, Bucket/binary,
   %% The resolution of data in this bucket.
     Resolution:?TIME_SIZE/?TIME_TYPE,
   %% The points per file in this bucket
     PPF:?TIME_SIZE/?TIME_TYPE,
   %% The time a bucket will retain data, 0 indicates indefinite
     TTL:?TIME_SIZE/?TIME_TYPE
   >>.

Deleting a bucket
-----------------

Deletes a bucket from the system.

.. warning::

   Not yet implemented.

.. code-block:: erlang

   <<9,
   %% The Size of the bucket binary and the bucket itself
     BucketSize:?BUCKET_SS/?SIZE_TYPE, Bucket:BucketSize/binary
   >>.
