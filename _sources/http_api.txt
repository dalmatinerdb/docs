.. DalmatinerDB HTTP API documentation, created by
   Heinz N. Gies on Sat Jul  7 16:49:03 2014.

HTTP API
========

The DalmatinerDB frontend offers a simplistic HTTP API for running queries along with a basic UI. All endpoints respond in regards of the **Content-Type** requested. Three types are currently valid:

* text/html - will return a human readable UI page.
* application/json - returns the result in json format.
* application/x-messagepack - returns the result's MessagePack encoded.

Using MessagePack is recommended as it both conserves computation time and bandwidth during encoding and decoding.

Bucket Listing
--------------

To list all buckets, submit a `GET` request to the endpoint `/buckets`. The reponse will return a list of all buckets known to DalmatinerDB.

Metric Listing
--------------

To list all the metrics in a bucket, submit a `GET` request to the endpoint `/buckets/<bucket name>`. The response will return a list of all metrics inside the requested bucket.

DQL Query
---------

To execute a DQL query a `GET` request is performed on the endpint `/`, and the query passed with the parameter `q`. The response will return an object of the form:

.. code-block:: javascript

   {
    "t": 1.2, // Duration in milliseconds
    "d": [{/*...*/}] // Result objects
   }

The result object looks like this:

.. code-block:: js

   {
    "n": "avg", // name of the result
    "r": 1, // resolution of the result
    "v": [42 /*, ...*/] // The array of values
   }
