.. DalmatinerDB documentation master file, created by
   Heinz N. Gies on Sat Jul  5 16:49:03 2014.

HTTP API
========

The dalmatiner frontend offers a simplistic HTTP API for running queries along with a basic UI. All endpoints respond in regards of the **Content-Type** requested. Three types are currently valid:

* text/html - will return a human redable UI page.
* application/json - returns the result in json format.
* application/x-messagepack - returns the results messagepack encoded.

Using msgpack is recommanded as it both conserves computation time and bandwith during en- and decoding.

Bucket Listing
--------------

To list buckets the endpoint `/buckets` is querried with a `GET` request, it will return a list of all buckets known to DalmatinerDB.

Metric Listing
--------------

Lists all the metrics in a bucket, the endpoint `/buckets/<bucket name>` is querried with a `GET` request, it will return a list of all metrics inside the requested bucket.

DQL Query
---------

To execute a DQL query a `GET` request is performed on the endpint `/`, and the query passed with the parameter `q`. Returned is a object of the form:

.. code-block:: javascript

   {
    "t": 1.2, // Duration in miliseconds
    "d": [{/*...*/}] // Result objects
   }

The result object looks like this:

.. code-block:: js

   {
    "n": "avg", // name of the result
    "r": 1, // resolution of the result
    "v": [42 /*, ...*/] // The array of values
   }
