.. DalmatinerDB installation manual
   sphinx-quickstart on Sat Jul  5 16:49:03 2014.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

Installation
============

Binaries
--------

Sorry at this point in time no bianries exist.


From Sorce
----------

Dependencies:

* Erlang > R16B3
* Make
* GCC

Installing the datastore
````````````````````````
.. code-block:: bash
   git clone https://github.com/dalmatinerdb/dalmatinerdb.git
   cd dalmatinerdb
   make deps all rel
   cp rel/dalmatinerdb $TARGET_DIRECTORY
   cd $TARGET_DIRECTORY
   cp etc/dalmatinerdb.conf.example etc/dalmatinerdb.conf
   vi etc/dalmatinerdb.conf # check the settings and adjust if needed
   ./bin/ddb start

Installing the frontend
```````````````````````

.. code-block:: bash
   git clone https://github.com/dalmatinerdb/dalmatiner-frontend.git
   cd dalmatiner-frontend
   make deps all rel
   cp rel/dalmatinerfe $TARGET_DIRECTORY
   cd $TARGET_DIRECTORY
   cp etc/dalmatinerfe.conf.example etc/dalmatinerfe.conf
   vi etc/dalmatinerfe.conf # check the settings and adjust if needed
   ./bin/dalmatinerfe start


 .. warning::
    At the moment automatic handling of disconnected upstream servers isn't handled well and might require a restart of the frontend.
