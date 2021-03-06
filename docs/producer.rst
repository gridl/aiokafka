.. _producer-usage:

Producer client
===============

.. _delivery semantics: https://kafka.apache.org/documentation/#semantics

:ref:`AIOKafkaProducer <aiokafka-producer>` is a client that publishes records
to the Kafka cluster. Most simple usage would be::

    producer = AIOKafkaProducer(
        loop=loop, bootstrap_servers='localhost:9092')
    await producer.start()
    try:
        await producer.send_and_wait("my_topic", b"Super message")
    finally:
        await producer.stop()

Under the hood **Producer** does quite some work on message delivery including
batching, retries, etc. All of it can be configured, so lets go through some
components for better understanding of the configuration options.


Buffer space
------------

While the user would expect the example above to send ``"Super message"``
directly to the broker, it's actually not sent right away, but rather added to
a **buffer space**. A background task will then get batches of messages and
send them to appropriate nodes in the cluster. This batching schema allows
*more throughput and more efficient compression*. To see it more clearly lets
avoid ``send_and_wait`` shortcut::

    # Will add message to 1st partition's batch. If this method times out, we
    # can say for sure that message was not ever sent.
    fut = await producer.send("my_topic", b"Super message", partition=1)

    # Here message will either be delivered or an unrecoverable error will
    # occure.
    # Cancelling this future will not cancel the send.
    msg = await fut


Batches themselves are created **per partition** with a maximum size of
``max_batch_size``. Messages in a batch are strictly in append order and only
1 batch per partition can be sent at a time (*aiokafka* does not support
``max.inflight.requests.per.connection`` option present in Java client). This
make a strict guarantee on message order in a partition.

By default the batch is sent immediately after the previous one (even if it's
not full). If you want to reduce the number of requests you can set
``linger_ms`` to something other than 0. This will add an additional delay
before sending next batch of requests (but only if it's not full).

``aiokafka`` does not (yet!) support some options, supported by Java's client:

    * ``buffer.memory`` to limit how much buffer space is used by producer to
      schedule requests in *all partitions*.
    * ``max.block.ms`` to limit the amount of time ``send()`` coroutine will
      wait for buffer append, when memory limit is reached. For now use::

        await asyncio.wait_for(producer.send(...), timeout=timeout)


Retries and Message acknowledgement
-----------------------------------

*aiokafka* will retry most errors automatically, but only until
``request_timeout_ms``. If request is expired, the last error will be raised
to the application. Retrying messages on application level after an error
will potentially lead to duplicates, so it's up to the user to decide.

For example, if ``RequestTimedOutError`` is raised, Producer can't be sure if
the Broker wrote the request or not.

The ``acks`` option controlls when the produce request is considered
acknowledged.

The most durable is ``acks="all"`` setting when Broker will wait for all
available replicas to write the request before replying to Producer. Broker
will consult it's ``min.insync.replicas`` setting to know the minimal amount of
replicas to write. If there's not enough in sync replicas either
``NotEnoughReplicasError`` or ``NotEnoughReplicasAfterAppendError`` will be
raised. It's up to the user what to do in those cases, as the errors are not
retriable.

The default is ``ack=1`` setting. It will not wait for relica writes, only for
Leader to write the request.

The least safe is ``ack=0`` when there will be no acknowledgement from Broker,
meaning client will never retry, as it will never see any errors.


Returned RecordMetadata object
------------------------------

After message is sent the user receives a ``RecordMetadata`` object containing
fields:

    * ``offset`` - unique offset of message in this partition. See 
      :ref:`Offsets and Consumer Position <offset_and_position>` for
      more details on offsets.
    * ``topic`` - *string* topic name
    * ``partition`` - *int* partition number

