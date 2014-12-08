.. DalmatinerDB data input manual
   Heinz N. Gies on Sat Jul  5 16:49:03 2014.

Data Input
==========

DalmatinerDB uses both UDP and TCP for data ingress allowing users to choose the transport most suited for their needs. The package structure is entirely the same the only difference is that `TCP packages <tcp_proto.html>`_ are prefixed with a 4 byte size.

Metric Package
--------------

Metrics are sent as size prefixed data. The layout of a metric package looks like this:

.. code-block:: erlang

   <<0,                                   %% Prefix to denote type of message
     Time:64/integer,                     %% The time (or offset) of the package
     BucketSize:16:integer,               %% The size of the bucket name
     Bucket:BucketSize/binary             %% The bucket to write the metric to
     MetricSection/binary                 %% The One or more metric entries
     >>



All sizes are given in bytes. The values are unsigned integers in **network byte order**. Especially with `DataSize` this is to be noted since it does **NOT** reflect the number of datapoints but rather the number of bytes used, thus it has to be a multiple of 9. As a result, a maximum of 7281 datapoints can be sent per metric package, not 65536.

Metric Section
--------------

The metric section can consist out of one or more metric blocks as described below, blocks are simply concatted sunce UDP packages have a fixed size, prefixing with size is not required.

.. code-block:: erlang

   <<MetricSize:16:integer,               %% The size of the metric name
     Metric:MetricSize/binary             %% The metric name
     DataSize:16:integer,                 %% The size of the metric name
     Data:DataSize/binary                 %% The metric name
     >>

Data Section
------------

The data section contains raw data for the database, each datapoint consists out of 9 bytes. 1 byte for indicating written data and 8 byte for a 64 bit signed integer in network byte order.

.. code-block:: erlang

   <<1, Value:64/signed-integer>>
