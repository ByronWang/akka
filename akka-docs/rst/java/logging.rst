.. _logging-java:

################
 Logging
################

Logging in Akka is not tied to a specific logging backend. By default
log messages are printed to STDOUT, but you can plug-in a SLF4J logger or 
your own logger. Logging is performed asynchronously to ensure that logging
has minimal performance impact. Logging generally means IO and locks,
which can slow down the operations of your code if it was performed 
synchronously.

How to Log
==========

Create a ``LoggingAdapter`` and use the ``error``, ``warning``, ``info``, or ``debug`` methods,
as illustrated in this example:

.. includecode:: code/docs/event/LoggingDocTest.java
   :include: imports

.. includecode:: code/docs/event/LoggingDocTest.java
   :include: my-actor

The first parameter to ``Logging.getLogger`` could also be any
:class:`LoggingBus`, specifically ``system.eventStream()``; in the demonstrated
case, the actor system’s address is included in the ``akkaSource``
representation of the log source (see `Logging Thread, Akka Source and Actor System in MDC`_)
while in the second case this is not automatically done. The second parameter
to ``Logging.getLogger`` is the source of this logging channel.  The source
object is translated to a String according to the following rules:

  * if it is an Actor or ActorRef, its path is used
  * in case of a String it is used as is
  * in case of a class an approximation of its simpleName
  * and in all other cases the simpleName of its class

The log message may contain argument placeholders ``{}``, which will be
substituted if the log level is enabled. Giving more arguments than
placeholders results in a warning being appended to the log statement (i.e. on
the same line with the same severity). You may pass a Java array as the only
substitution argument to have its elements be treated individually:

.. includecode:: code/docs/event/LoggingDocTest.java#array

The Java :class:`Class` of the log source is also included in the generated
:class:`LogEvent`. In case of a simple string this is replaced with a “marker”
class :class:`akka.event.DummyClassForStringSources` in order to allow special
treatment of this case, e.g. in the SLF4J event listener which will then use
the string instead of the class’ name for looking up the logger instance to
use.

Logging of Dead Letters
-----------------------

By default messages sent to dead letters are logged at info level. Existence of dead letters
does not necessarily indicate a problem, but they are logged by default for the sake of caution.
After a few messages this logging is turned off, to avoid flooding the logs.
You can disable this logging completely or adjust how many dead letters are
logged. During system shutdown it is likely that you see dead letters, since pending
messages in the actor mailboxes are sent to dead letters. You can also disable logging
of dead letters during shutdown.

.. code-block:: ruby

    akka {
      log-dead-letters = 10
      log-dead-letters-during-shutdown = on
    }

To customize the logging further or take other actions for dead letters you can subscribe
to the :ref:`event-stream-java`.

Auxiliary logging options
-------------------------

Akka has a few configuration options for very low level debugging. These make more sense in development than in production. 

You almost definitely need to have logging set to DEBUG to use any of the options below:

.. code-block:: ruby

    akka {
      loglevel = "DEBUG"
    }

This config option is very good if you want to know what config settings are loaded by Akka:

.. code-block:: ruby

    akka {
      # Log the complete configuration at INFO level when the actor system is started.
      # This is useful when you are uncertain of what configuration is used.
      log-config-on-start = on
    }

If you want very detailed logging of all automatically received messages that are processed
by Actors:

.. code-block:: ruby

    akka {
      actor {
        debug {
          # enable DEBUG logging of all AutoReceiveMessages (Kill, PoisonPill et.c.)
          autoreceive = on
        }
      }
    }

If you want very detailed logging of all lifecycle changes of Actors (restarts, deaths etc):

.. code-block:: ruby

    akka {
      actor {
        debug {
          # enable DEBUG logging of actor lifecycle changes
          lifecycle = on
        }
      }
    }

If you want unhandled messages logged at DEBUG:

.. code-block:: ruby

    akka {
      actor {
        debug {
          # enable DEBUG logging of unhandled messages
          unhandled = on
        }
      }
    }

If you want very detailed logging of all events, transitions and timers of FSM Actors that extend LoggingFSM:

.. code-block:: ruby

    akka {
      actor {
        debug {
          # enable DEBUG logging of all LoggingFSMs for events, transitions and timers
          fsm = on
        }
      }
    }

If you want to monitor subscriptions (subscribe/unsubscribe) on the ActorSystem.eventStream:

.. code-block:: ruby

    akka {
      actor {
        debug {
          # enable DEBUG logging of subscription changes on the eventStream
          event-stream = on
        }
      }
    }

.. _logging-remote-java:

Auxiliary remote logging options
--------------------------------

If you want to see all messages that are sent through remoting at DEBUG log level, use the following config option. Note that this logs the messages as they are sent by the transport layer, not by an actor.

.. code-block:: ruby

    akka {
      remote {
        # If this is "on", Akka will log all outbound messages at DEBUG level,
        # if off then they are not logged
        log-sent-messages = on
      }
    }

If you want to see all messages that are received through remoting at DEBUG log level, use the following config option. Note that this logs the messages as they are sent by the transport layer, not by an actor.

.. code-block:: ruby

    akka {
      remote {
        # If this is "on", Akka will log all inbound messages at DEBUG level,
        # if off then they are not logged
        log-received-messages = on
      }
    }

If you want to see message types with payload size in bytes larger than
a specified limit at INFO log level:

.. code-block:: ruby

    akka {
      remote {
        # Logging of message types with payload size in bytes larger than
        # this value. Maximum detected size per message type is logged once,
        # with an increase threshold of 10%.
        # By default this feature is turned off. Activate it by setting the property to
        # a value in bytes, such as 1000b. Note that for all messages larger than this
        # limit there will be extra performance and scalability cost.
        log-frame-size-exceeding = 1000b
      }
    }

Also see the logging options for TestKit: :ref:`actor.logging-java`.

Turn Off Logging
----------------

To turn off logging you can configure the log levels to be ``OFF`` like this.

.. code-block:: ruby

  akka {
    stdout-loglevel = "OFF"
    loglevel = "OFF"
  }

The ``stdout-loglevel`` is only in effect during system startup and shutdown, and setting
it to ``OFF`` as well, ensures that nothing gets logged during system startup or shutdown.

Loggers
=======

Logging is performed asynchronously through an event bus. Log events are processed by an event handler actor
that receives the log events in the same order they were emitted.

.. note::
  The event handler actor does not have a bounded inbox and is run on the default dispatcher. This means
  that logging extreme amounts of data may affect your application badly. This can be somewhat mitigated by using an async logging backend though. (See :ref:`slf4j-directly-java`)

You can configure which event handlers are created at system start-up and listen to logging events. That is done using the 
``loggers`` element in the :ref:`configuration`.
Here you can also define the log level. More fine grained filtering based on the log source 
can be implemented in a custom ``LoggingFilter``, which can be defined in the ``logging-filter`` 
configuration property. 

.. code-block:: ruby

  akka {
    # Loggers to register at boot time (akka.event.Logging$DefaultLogger logs
    # to STDOUT)
    loggers = ["akka.event.Logging$DefaultLogger"]
    # Options: OFF, ERROR, WARNING, INFO, DEBUG
    loglevel = "DEBUG"
  }

The default one logs to STDOUT and is registered by default. It is not intended to be used for production. There is also an :ref:`slf4j-java`
logger available in the 'akka-slf4j' module.

Example of creating a listener:

.. includecode:: code/docs/event/LoggingDocTest.java
   :include: imports,imports-listener

.. includecode:: code/docs/event/LoggingDocTest.java
   :include: my-event-listener

Logging to stdout during startup and shutdown
=============================================

While the actor system is starting up and shutting down the configured ``loggers`` are not used.
Instead log messages are printed to stdout (System.out). The default log level for this
stdout logger is ``WARNING`` and it can be silenced completely by setting 
``akka.stdout-loglevel=OFF``.

.. _slf4j-java:

SLF4J
=====

Akka provides a logger for `SL4FJ <http://www.slf4j.org/>`_. This module is available in the 'akka-slf4j.jar'.
It has a single dependency: the slf4j-api jar. In your runtime, you also need a SLF4J backend. We recommend `Logback <http://logback.qos.ch/>`_:

  .. code-block:: xml

     <dependency>
       <groupId>ch.qos.logback</groupId>
       <artifactId>logback-classic</artifactId>
       <version>1.1.3</version>
     </dependency>

You need to enable the Slf4jLogger in the ``loggers`` element in
the :ref:`configuration`. Here you can also define the log level of the event bus.
More fine grained log levels can be defined in the configuration of the SLF4J backend
(e.g. logback.xml). You should also define ``akka.event.slf4j.Slf4jLoggingFilter`` in
the ``logging-filter`` configuration property. It will filter the log events using the backend
configuration (e.g. logback.xml) before they are published to the event bus.

.. warning::
  If you set the ``loglevel`` to a higher level than "DEBUG", any DEBUG events will be filtered
  out already at the source and will never reach the logging backend, regardless of how the backend
  is configured.

.. code-block:: ruby

  akka {
    loggers = ["akka.event.slf4j.Slf4jLogger"]
    loglevel = "DEBUG"
    logging-filter = "akka.event.slf4j.Slf4jLoggingFilter"
  }
  
One gotcha is that the timestamp is attributed in the event handler, not when actually doing the logging.

The SLF4J logger selected for each log event is chosen based on the
:class:`Class` of the log source specified when creating the
:class:`LoggingAdapter`, unless that was given directly as a string in which
case that string is used (i.e. ``LoggerFactory.getLogger(Class c)`` is used in
the first case and ``LoggerFactory.getLogger(String s)`` in the second).

.. note::

  Beware that the actor system’s name is appended to a :class:`String` log
  source if the LoggingAdapter was created giving an :class:`ActorSystem` to
  the factory. If this is not intended, give a :class:`LoggingBus` instead as
  shown below:

.. code-block:: scala

  final LoggingAdapter log = Logging.getLogger(system.eventStream(), "my.string");

.. _slf4j-directly-java:

Using the SLF4J API directly
----------------------------
If you use the SLF4J API directly in your application, remember that the logging operations will block
while the underlying infrastructure writes the log statements.

This can be avoided by configuring the logging implementation to use
a non-blocking appender. Logback provides `AsyncAppender <http://logback.qos.ch/manual/appenders.html#AsyncAppender>`_
that does this. It also contains a feature which will drop ``INFO`` and ``DEBUG`` messages if the logging
load is high.

Logging Thread, Akka Source and Actor System in MDC
---------------------------------------------------

Since the logging is done asynchronously the thread in which the logging was performed is captured in
Mapped Diagnostic Context (MDC) with attribute name ``sourceThread``.
With Logback the thread name is available with ``%X{sourceThread}`` specifier within the pattern layout configuration::

  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%date{ISO8601} %-5level %logger{36} %X{sourceThread} - %msg%n</pattern>
    </encoder>
  </appender>

.. note::

  It will probably be a good idea to use the ``sourceThread`` MDC value also in
  non-Akka parts of the application in order to have this property consistently
  available in the logs.

Another helpful facility is that Akka captures the actor’s address when
instantiating a logger within it, meaning that the full instance identification
is available for associating log messages e.g. with members of a router. This
information is available in the MDC with attribute name ``akkaSource``::

  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%date{ISO8601} %-5level %logger{36} %X{akkaSource} - %msg%n</pattern>
    </encoder>
  </appender>

Finally, the actor system in which the logging was performed
is available in the MDC with attribute name ``sourceActorSystem``::

  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%date{ISO8601} %-5level %logger{36} %X{sourceActorSystem} - %msg%n</pattern>
    </encoder>
  </appender>

For more details on what this attribute contains—also for non-actors—please see
`How to Log`_.

More accurate timestamps for log output in MDC
----------------------------------------------

Akka's logging is asynchronous which means that the timestamp of a log entry is taken from
when the underlying logger implementation is called, which can be surprising at first.
If you want to more accurately output the timestamp, use the MDC attribute ``akkaTimestamp``::

  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%X{akkaTimestamp} %-5level %logger{36} %X{akkaSource} - %msg%n</pattern>
    </encoder>
  </appender>


MDC values defined by the application
-------------------------------------

One useful feature available in Slf4j is `MDC <http://logback.qos.ch/manual/mdc.html>`_,
Akka has a way to let the application specify custom values, you just need to get a
specialized :class:`LoggingAdapter`, the :class:`DiagnosticLoggingAdapter`. In order to
get it you can use the factory, providing an AbstractActor as logSource:

.. code-block:: scala

    // Within your AbstractActor
    final DiagnosticLoggingAdapter log = Logging.getLogger(this);

Once you have the logger, you just need to add the custom values before you log something.
This way, the values will be put in the SLF4J MDC right before appending the log and removed after.

.. note::

  The cleanup (removal) should be done in the actor at the end,
  otherwise, the next message will log with same MDC values,
  if it is not set to a new map. Use ``log.clearMDC()``.

.. includecode:: code/docs/event/LoggingDocTest.java
    :include: imports-mdc

.. includecode:: code/docs/event/LoggingDocTest.java
    :include: mdc-actor

Now, the values will be available in the MDC, so you can use them in the layout pattern::

  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>
        %-5level %logger{36} [req: %X{requestId}, visitor: %X{visitorId}] - %msg%n
      </pattern>
    </encoder>
  </appender>


Using Markers
-------------

Some logging libraries allow, in addition to MDC data, attaching so called "markers" to log statements.
These are used to filter out rare and special events, for example you might want to mark logs that detect
some malicious activity and mark them with a ``SECURITY`` tag, and in your appender configuration make these
trigger emails and other notifications immediately.

Markers are available through the LoggingAdapters, when obtained via ``Logging.withMarker``.
The first argument passed into all log calls then should be a ``akka.event.LogMarker``. 

The slf4j bridge provided by akka in ``akka-slf4j`` will automatically pick up this marker value and make it available to SLF4J.
For example you could use it like this::

  <pattern>%date{ISO8601} [%marker][%level] [%msg]%n</pattern>

A more advanced (including most Akka added information) example pattern would be::

  <pattern>%date{ISO8601} level=[%level] marker=[%marker] logger=[%logger] akkaSource=[%X{akkaSource}] sourceActorSystem=[%X{sourceActorSystem}] sourceThread=[%X{sourceThread}] mdc=[ticket-#%X{ticketNumber}: %X{ticketDesc}] - msg=[%msg]%n----%n</pattern>

.. _jul-java:

java.util.logging
=================

Akka includes a logger for `java.util.logging <https://docs.oracle.com/javase/8/docs/api/java/util/logging/package-summary.html#package.description>`_.

You need to enable the ``akka.event.jul.JavaLogger`` in the ``loggers`` element in
the :ref:`configuration`. Here you can also define the log level of the event bus.
More fine grained log levels can be defined in the configuration of the logging backend. 
You should also define ``akka.event.jul.JavaLoggingFilter`` in
the ``logging-filter`` configuration property. It will filter the log events using the backend
configuration before they are published to the event bus.

.. warning::
  If you set the ``loglevel`` to a higher level than "DEBUG", any DEBUG events will be filtered
  out already at the source and will never reach the logging backend, regardless of how the backend
  is configured.

.. code-block:: ruby

  akka {
    loglevel = DEBUG
    loggers = ["akka.event.jul.JavaLogger"]
    logging-filter = "akka.event.jul.JavaLoggingFilter"
  }

One gotcha is that the timestamp is attributed in the event handler, not when actually doing the logging.

The ``java.util.logging.Logger`` selected for each log event is chosen based on the
:class:`Class` of the log source specified when creating the
:class:`LoggingAdapter`, unless that was given directly as a string in which
case that string is used (i.e. ``LoggerFactory.getLogger(Class c)`` is used in
the first case and ``LoggerFactory.getLogger(String s)`` in the second).

.. note::

  Beware that the actor system’s name is appended to a :class:`String` log
  source if the LoggingAdapter was created giving an :class:`ActorSystem` to
  the factory. If this is not intended, give a :class:`LoggingBus` instead as
  shown below:

.. code-block:: scala

  final LoggingAdapter log = Logging.getLogger(system.eventStream(), "my.string");
