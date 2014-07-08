.. DalmatinerDB data input manual
   Heinz N. Gies on Sat Jul  5 16:49:03 2014.

Data Input
==========

DalmatinerDB uses UDP for data ingres, which is one of its trade-offs. UDP is a lot faster than TCP. It prevents the providers from blocking and the consumer from overloading by dropping packages it can not handle. It is possible to have multiple UDP ports for increased concurrency.

Metric Package
--------------

Metrics are sent as size prefixed data. The layout of a metric package looks like this:

.. code-block:: erlang

   <<0,                                   %% Prefix to denote type of message
     Time:64/integer,                     %% The time (or offset) of the package
     BucketSize:16:integer,               %% The size of the bucket name
     Bucket:BucketSize/binary             %% The bucket to write the metric to
     MetricSize:16:integer,               %% The size of the metric name
     Metric:MetricSize/binary             %% The metric name
     DataSize:16:integer,                 %% The size of the metric name
     Data:DataSize/binary                 %% The metric name
     >>

All sizes are given in bytes. The values are unsigned integers in **network byte order**. Especially with `DataSize` this is to be noted since it does **NOT** reflect the number of datapoints but rather the number of bytes used, thus it has to be a multiple of 9. As a result, a maximum of 7281 datapoints can be sent per metric package, not 65536.

Not only can a metric package have multiple consecutive datapoints, it is also possible to combine multiple metric packages into a single UDP datagram. This allows to set multiple different metrics at once. Combining metric packages allows for some optimizations and is recommended as long as the liveliness of the data doesn't prevent it.

Data Section
------------

The data section contains raw data for the database, each datapoint consists out of 9 bytes. 1 byte for indicating written data and 8 byte for a 64 bit signed integer in network byte order.


.. code-block:: erlang

   <<1, Value:64/signed-integer>>
