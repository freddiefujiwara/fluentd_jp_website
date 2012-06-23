.. _overview:

Overview
========================

**Fluentd** is a log collector daemon, which treats logs as JSON stream. So far, the largest user collects logs from 100+ servers, 650 GB daily, with 70,000 msgs/sec at peak time.

Purpose
-------

Modern web and mobile applications generate a very large number of **event logs** (ex: login, logout, purchase, follow, etc). By analyzing these event logs, these services can be improved greatly. However, collecting these logs in a simple and reliable manner remains a challenge. 

Fluentd solves this problem by providing the user with the following features:

* Easy Installation
* Small Footprint
* Semi-Structured Data Logging
* Flexible Plugin Mechanism
* Reliable Buffering
* Log Forwarding

Easy Installation
-----------------

**Fluentd** is packaged as a Ruby gem. It can be installed with a single command.

Small Footprint
---------------

Due to its simple architecture, Fluentd’s core consists of just 3,000 lines of Ruby. Fluentd collects events from various **input** sources and writes them to **output** sinks.

* Input Examples: HTTP, Syslog, Apache Log
* Output Examples: Files, Mails, RDBMS databases, NoSQL storages

The figure below shows the basic idea of **input** and **output**::

        Input                          Output
    +--------------------------------------------+
    |                                            |
    |  Web Apps  ---+                 +--> File  |
    |               |                 |          |
    |               +-->           ---+          |
    |  /var/log  ------>  Fluentd  ------> Mail  |
    |               +-->           ---+          |
    |               |                 |          |
    |  Apache    ---+                 +--> S3    |
    |                                            |
    +--------------------------------------------+

Semi-Structured Data Logging
----------------------------

A collected event log consists of three entities: *tag*, *time* and *record*. Tag is a string separated by '.' (e.g. myapp.access), and is used to categorize events. Time is the UNIX time when the event occurs. Record is a JSON object.

Flexible Plugin Mechanism
-------------------------

Fluentd’s inputs sources and output destinations can be extended by writing appropriate Ruby plugins; they can then be published via Ruby gems. The list of available plugins can be seen with the following command::

  $ gem search -rd fluent-plugin

Reliable Buffering
-------------------

In traditional systems, event logs can be lost when an unexpected output write failure (ex: network failure) occurs. Fluentd has been designed to combat this issue, and is equipped with a reliable buffering strategy.  Fluentd’s buffer, a queue of chunks containing the event logs, temporarily stores the collected events::

    Queue
    +---------+
    |         |
    |  Chunk <-- Write events to the top chunk
    |         |  (never block)
    |  Chunk  |
    |         |
    |  Chunk  |
    |         |
    |  Chunk --> Write out the bottom chunk
    |         |  (transactional)
    +---------+

When Fluentd receives an event from its input source, the event log is appended to the top chunk in the buffer. This operation for temporary storage is never blocked, even if the next server is down.

A new empty chunk is pushed onto the top of the queue when either (1) the size of the top chunk reaches its limit, or (2) the timer expires. 

A separate thread writes the bottom chunk out to either the next server or the storage server. If this write operation is successful, the chunk is removed from the queue. Otherwise, the thread leaves the chunk in the queue and will try again later.

Fluentd’s buffer implementation is pluggable. The default plugin, 'Memory', stores the chunks in memory. It is fast but not persistent. Another plugin, 'File', stores the chunks in file.

Log Forwarding
--------------

Fluentd supports both single-node and multi-node configurations. In multi-node configuration, Fluentd supports log forwarding in order to collect the event logs into one location for analysis. Application servers will forward the local logs of their Fluentd instances to the Fluentd instance of a central server::

    Web Server
    +---------+
    | Fluentd -------+
    +---------+      |
                     |
    Proxy Server     |
    +---------+      +--> +---------+
    | Fluentd ----------> | Fluentd |
    +---------+      +--> +---------+
                     |
    Database Server  |
    +---------+      |
    | Fluentd -------+
    +---------+

Next step: :ref:`install`
