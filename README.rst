######
graypy_v6
######

Description
===========

Python logging handlers that send log messages in the
Graylog Extended Log Format (GELF_).

graypy_v6 supports sending GELF logs to both Graylog2 and Graylog3 servers with IPv6 protocol.

Installing
==========

Using pip
---------

Install the basic graypy python logging handlers:

.. code-block:: console

    pip install -e git://github.com/covx/graypy_v6.git@3.0.1#egg=graypy

Usage
=====

graypy sends GELF logs to a Graylog server via subclasses of the python
`logging.Handler`_ class.

Below is the list of ready to run GELF logging handlers defined by graypy:

* ``GELFUDPHandler`` - UDP log forwarding
* ``GELFUDPIPv6Handler`` - UDP log forwarding with IPv6 protocol
* ``GELFTCPHandler`` - TCP log forwarding
* ``GELFTLSHandler`` - TCP log forwarding with TLS support
* ``GELFHTTPHandler`` - HTTP log forwarding
* ``GELFRabbitHandler`` - RabbitMQ log forwarding

UDP Logging
-----------

UDP Log forwarding to a locally hosted Graylog server can be easily done with
the ``GELFUDPHandler``:

.. code-block:: python

    import logging
    import graypy

    my_logger = logging.getLogger('test_logger')
    my_logger.setLevel(logging.DEBUG)

    handler = graypy.GELFUDPHandler('localhost', 12201)
    my_logger.addHandler(handler)

    my_logger.debug('Hello Graylog.')


UDP GELF Chunkers
^^^^^^^^^^^^^^^^^

`GELF UDP Chunking`_ is supported by the ``GELFUDPHandler`` and is defined by
the ``gelf_chunker`` argument within its constructor. By default the
``GELFWarningChunker`` is used, thus, GELF messages that chunk overflow
(i.e. consisting of more than 128 chunks) will issue a
``GELFChunkOverflowWarning`` and **will be dropped**.

Other ``gelf_chunker`` options are also available:

* ``BaseGELFChunker`` silently drops GELF messages that chunk overflow
* ``GELFTruncatingChunker`` issues a ``GELFChunkOverflowWarning`` and
  simplifies and truncates GELF messages that chunk overflow in a attempt
  to send some content to Graylog. If this process fails to prevent
  another chunk overflow a ``GELFTruncationFailureWarning`` is issued.

RabbitMQ Logging
----------------

Alternately, use ``GELFRabbitHandler`` to send messages to RabbitMQ and
configure your Graylog server to consume messages via AMQP. This prevents log
messages from being lost due to dropped UDP packets (``GELFUDPHandler`` sends
messages to Graylog using UDP). You will need to configure RabbitMQ with a
``gelf_log`` queue and bind it to the ``logging.gelf`` exchange so messages
are properly routed to a queue that can be consumed by Graylog (the queue and
exchange names may be customized to your liking).

.. code-block:: python

    import logging
    import graypy

    my_logger = logging.getLogger('test_logger')
    my_logger.setLevel(logging.DEBUG)

    handler = graypy.GELFRabbitHandler('amqp://guest:guest@localhost/', exchange='logging.gelf')
    my_logger.addHandler(handler)

    my_logger.debug('Hello Graylog.')

Django Logging
--------------

It's easy to integrate ``graypy`` with Django's logging settings. Just add a
new handler in your ``settings.py``:

.. code-block:: python

    LOGGING = {
        'version': 1,
        # other dictConfig keys here...
        'handlers': {
            'graypy': {
                'level': 'WARNING',
                'class': 'graypy.GELFUDPHandler',
                'host': 'localhost',
                'port': 12201,
            },
        },
        'loggers': {
            'django.request': {
                'handlers': ['graypy'],
                'level': 'ERROR',
                'propagate': True,
            },
        },
    }

Django Logging with IPv6
------------------------

It's easy to integrate ``graypy_v6`` with IPv6 Django's logging settings. Just add a
new handler in your ``settings.py``:

.. code-block:: python

    LOGGING = {
        'version': 1,
        # other dictConfig keys here...
        'handlers': {
            'graypy': {
                'level': 'WARNING',
                'class': 'graypy.GELFUDPIPv6Handler',
                'host': '[::1]',
                'port': 12201,
            },
        },
        'loggers': {
            'django.request': {
                'handlers': ['graypy'],
                'level': 'ERROR',
                'propagate': True,
            },
        },
    }

Traceback Logging
-----------------

By default log captured exception tracebacks are added to the GELF log as
``full_message`` fields:

.. code-block:: python

    import logging
    import graypy

    my_logger = logging.getLogger('test_logger')
    my_logger.setLevel(logging.DEBUG)

    handler = graypy.GELFUDPHandler('localhost', 12201)
    my_logger.addHandler(handler)

    try:
        puff_the_magic_dragon()
    except NameError:
        my_logger.debug('No dragons here.', exc_info=1)

Default Logging Fields
----------------------

By default a number of debugging logging fields are automatically added to the
GELF log if available:

    * function
    * pid
    * process_name
    * thread_name

You can disable automatically adding these debugging logging fields by
specifying ``debugging_fields=False`` in the handler's constructor:

.. code-block:: python

    handler = graypy.GELFUDPHandler('localhost', 12201, debugging_fields=False)

Adding Custom Logging Fields
----------------------------

graypy also supports including custom fields in the GELF logs sent to Graylog.
This can be done by using Python's LoggerAdapter_ and Filter_ classes.

Using LoggerAdapter
^^^^^^^^^^^^^^^^^^^

LoggerAdapter_ makes it easy to add static information to your GELF log
messages:

.. code-block:: python

    import logging
    import graypy

    my_logger = logging.getLogger('test_logger')
    my_logger.setLevel(logging.DEBUG)

    handler = graypy.GELFUDPHandler('localhost', 12201)
    my_logger.addHandler(handler)

    my_adapter = logging.LoggerAdapter(logging.getLogger('test_logger'),
                                       {'username': 'John'})

    my_adapter.debug('Hello Graylog from John.')

Using Filter
^^^^^^^^^^^^

Filter_ gives more flexibility and allows for dynamic information to be
added to your GELF logs:

.. code-block:: python

    import logging
    import graypy

    class UsernameFilter(logging.Filter):
        def __init__(self):
            # In an actual use case would dynamically get this
            # (e.g. from memcache)
            self.username = 'John'

        def filter(self, record):
            record.username = self.username
            return True

    my_logger = logging.getLogger('test_logger')
    my_logger.setLevel(logging.DEBUG)

    handler = graypy.GELFUDPHandler('localhost', 12201)
    my_logger.addHandler(handler)

    my_logger.addFilter(UsernameFilter())

    my_logger.debug('Hello Graylog from John.')

Contributors
============

  * Sever Banesiu
  * Daniel Miller
  * Tushar Makkar
  * Nathan Klapstein
  * Maxim Chernyatevich

.. _GELF: https://docs.graylog.org/en/latest/pages/gelf.html
.. _logging.Handler: https://docs.python.org/3/library/logging.html#logging.Handler
.. _GELF UDP Chunking: https://docs.graylog.org/en/latest/pages/gelf.html#chunking
.. _LoggerAdapter: https://docs.python.org/howto/logging-cookbook.html#using-loggeradapters-to-impart-contextual-information
.. _Filter: https://docs.python.org/howto/logging-cookbook.html#using-filters-to-impart-contextual-information
