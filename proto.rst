.. DalmatinerDB data input manual
   Heinz N. Gies on Sat June 5 16:49:03 2014.


Network Protocol
****************

General
=======

Constatns / Macros
------------------
The following constants (Macros) are defined as part of the protocol as they name frequently reoccuring values:

.. code-block:: erlang

   %% dproto.hrl

   %% number of bits used to encode the bucket size.
   %% => buckets can be 255 byte at most!
   -define(BUCKET_SS, 8).

   %% The number of bits used to encode the size of the a single metric part.
   %% => each part can be 255 byte at max!
   -define(METRIC_ELEMENT_SS, 8).


   %% The number of bits used to encode the size of a whole metric (All of it's
   %% parts)
   %% => the maximum number of bytes in a whole metric can be 65,536, so a single
   %% metric can hold at least 255 elments, more if their size is < 256 byte.
   -define(METRIC_SS, 16).

   %% The number of bits used to encode a list of metrics.
   %% => this means a list opperation can return at least 281,474,976,710,656
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

Metric names are not simple strings but a lenght prefixed list of elements. The upside of this is that there are no reserved characters (such as ``.``) and it allows a lot faster parsing and matching against them.

An example would be:

.. code-block:: erlang

   <<2, "my", 3, "key">>.
   <<3, "yet", 7, "another", 3, "one">>.

TCP
===

All TCP data is prefixed with 4 byte size.


Ingress
-------

The TCP endpoint can only accept incoming data when switched to stream mode. This way a connetion is dedicated to send data to a single bucket. Flushing can be handled either manually or automatically. Automatic flushing sets a maximum delta between the first data cached for the connection the newest arrived bit of information.


Stream Mode
```````````

It is possible to swtich the TCP connection, this stream allows to specify a bucket for the stream
and by that prevent it to be resend with every metric. Also it makes it possible for the connection
cache to have a specified maximal duration between the first and the last metric received before the
data is flushed.

Initializing
''''''''''''

This will switch the TCP connection to stream mode from now on only payload and flush messages
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


Payload
'''''''

The metric packages automatically flash the connection cache when ``(Time - min(All Times)) > MaxDelay``.

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
'''''

It is possible to control the flush time outside of the timing by forcing a flush as part of the stream. To do that the ``flush`` message can be used.

.. code-block:: erlang

   <<6>>. % Indicates that at this point the connection cache should be flushed.

Querying
--------

List Buckets
````````````

This command list all buckets, each bucket known to the system. The command is received and a reply send directly.

.. code-block:: erlang

   <<3>>.

The Reply is prefixed with the total size of the whole reply in bytes (not including the size prefix itself). Then each bucket is prefixed by a size of the bucket name.

.. code-block:: erlang

   %% Outer wrapper
   <<ReplySize:?BUCKETS_SS/?SIZE_TYPE, Reply:ReplySize/binary>>.
   %% Elements of the reply
   <<BucketSize:?BUCKET_SS/?SIZE_TYPE, Bucket:BucketSize/binary>>.


List Metrics
````````````

Lists all metrics in a bucket. The bucket to look for is prefixed by 1 byte size for the bucket name.

.. code-block:: erlang

   <<1,
   %% The size and the bucket binary to read the metric list from
     BucketSize:?BUCKET_SS/?SIZE_TYPE,
     Bucket:BucketSize/binary
   >>.

The Reply is prefixed with the total size of the whole reply in bytes (not including the size prefix itself). Then each metric is prefixed by a size of the metric name.

.. code-block:: erlang

   %% Outer wrapper
   <<ReplySize:32/integer, Reply:ReplySize/binary>>.
   %% Elements of the reply
   <<MetricSize:16/integer, Metric:MetricSize/binary>>.


Reading Data
````````````

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

There will **always** be returned ``Count`` messages will be returned, if there is no or insufficient data or the bucket/metric doesn't exist the missing data will be filled with blanks.

.. code-block:: erlang

   <<Reply:(?DATA_SIZE*Count)/binary>>.


where each of the elements looks like one of thise:


Bucket Information
``````````````````

Gets informations of the bucket, namely the resolution and the points per file.

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
----------

Adding a bucket
```````````````

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
`````````````````

Deletes a bucket from the system.

.. warning::

   Not yet implemented.

.. code-block:: erlang

   <<9,
   %% The Size of the bucket binary and the bucket itself
     BucketSize:?BUCKET_SS/?SIZE_TYPE, Bucket:BucketSize/binary
   >>.

UDP
===

Metric Package
--------------

Metrics are sent as size prefixed data. The layout of a metric package looks like this:

.. code-block:: erlang

   <<0,                                   %% Prefix to denote type of message
     BucketSize:?BUCKET_SS/?SIZE_TYPE,    %% The size of the bucket name
     Bucket:BucketSize/binary             %% The bucket to write the metric to
     MetricSection/binary                 %% The One or more metric entries
     >>


All sizes are given in bytes. The values are unsigned integers in **network byte order**. 

Metric Section
--------------

The metric section can consist out of one or more metric blocks as described below, blocks are simply concatted sunce UDP packages have a fixed size, prefixing with size is not required.

.. code-block:: erlang

   <<
     Time:?TIME_SIZE/?TIME_TYPE,          %% The time (or offset) of the package
     MetricSize:?METRIC_SS/?SIZE_TYPE,    %% The size of the metric name
     Metric:MetricSize/binary             %% The metric name
     DataSize:?DATA_SS/?SIZE_TYPE,        %% The size of the metric name
     Data:DataSize/binary                 %% The metric name
     >>

With ``DataSize`` this is to be noted since it does **NOT** reflect the number of datapoints but rather the number of bytes used, thus it has to be a multiple of ``?DATA_SIZE``. As a result, a maximum of 7281 datapoints can be sent per metric package, not 65536.

The Data section contains datapoints as documented above.
