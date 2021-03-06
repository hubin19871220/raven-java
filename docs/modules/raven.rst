Raven
=====

Main module of the Raven project in java. It provides a client to send
messages to a Sentry server as well as an implementation of an `Handler
<http://docs.oracle.com/javase/7/docs/api/java/util/logging/Handler.html>`_
for ``java.util.logging``.

The project can be found on github: `raven-java/raven
<https://github.com/getsentry/raven-java/tree/master/raven>`_

Installation
------------

If you want to use Maven you can install Raven as dependency:

.. sourcecode:: xml

    <dependency>
        <groupId>com.getsentry.raven</groupId>
        <artifactId>raven</artifactId>
        <version>7.6.0</version>
    </dependency>

If you manually want to manage your dependencies:

- `jackson-core-2.5.0.jar <https://search.maven.org/#artifactdetails%7Ccom.fasterxml.jackson.core%7Cjackson-core%7C2.5.0%7Cjar>`_
- `slf4j-api-1.7.9.jar <https://search.maven.org/#artifactdetails%7Corg.slf4j%7Cslf4j-api%7C1.7.9%7Cjar>`_
- `slf4j-jdk14-1.7.9.jar <https://search.maven.org/#artifactdetails%7Corg.slf4j%7Cslf4j-jdk14%7C1.7.9%7Cjar>`_
  is recommended as the implementation of slf4j to capture the log events
  generated by Raven (connection errors, ...) if `java.util.logging` is
  used.

Usage
-----

To use Raven you need to use it through ``java.util.logging``.  The
following configuration (``logging.properties``) gets you started:

.. sourcecode:: ini

    .level=WARN
    handlers=com.getsentry.raven.jul.SentryHandler
    com.getsentry.raven.jul.SentryHandler.dsn=___DSN___
    com.getsentry.raven.jul.SentryHandler.tags=tag1:value1,tag2:value2
    # Optional, allows to select the ravenFactory
    #com.getsentry.raven.jul.SentryHandler.ravenFactory=com.getsentry.raven.DefaultRavenFactory

When starting your application, add the ``java.util.logging.config.file`` to
the system properties, with the full path to the ``logging.properties`` as
its value::

    $ java -Djava.util.logging.config.file=/path/to/app.properties MyClass

Practical Example
-----------------

.. sourcecode:: java

    import java.util.logging.Level;
    import java.util.logging.Logger;

    public class MyClass {
        private static final Logger logger = Logger.getLogger(MyClass.class.getName());

        void logSimpleMessage() {
            // This adds a simple message to the logs
            logger.log(Level.INFO, "This is a test");
        }

        void logException() {
            try {
                unsafeMethod();
            } catch (Exception e) {
                // This adds an exception to the logs
                logger.log(Level.SEVERE, "Exception caught", e);
            }
        }

        void unsafeMethod() {
            throw new UnsupportedOperationException("You shouldn't call that");
        }
    }

``java.util.logging`` does not support either MDC nor NDC, meaning that it
is not possible to attach additional/custom context values to the logs. In
other terms, it is not possible to use the "extra" field supported by
Sentry.

Manual Usage
------------

It is possible to use the client manually rather than using a logging
framework in order to send messages to Sentry. It is not recommended to
use this solution as the API is more verbose and requires the developer to
specify the value of each field sent to Sentry:

.. sourcecode:: java

    import com.getsentry.raven.Raven;
    import com.getsentry.raven.RavenFactory;

    public class MyClass {
        private static Raven raven;

        public static void main(String... args) {
            // Creation of the client with a specific DSN
            String dsn = args[0];
            raven = RavenFactory.ravenInstance(dsn);

            // It is also possible to use the DSN detection system like this
            raven = RavenFactory.ravenInstance();
        }

        void logSimpleMessage() {
            // This adds a simple message to the logs
            raven.sendMessage("This is a test");
        }

        void logException() {
            try {
                unsafeMethod();
            } catch (Exception e) {
                // This adds an exception to the logs
                raven.sendException(e);
            }
        }

        void unsafeMethod() {
            throw new UnsupportedOperationException("You shouldn't call that");
        }
    }

For more complex messages, it will be necessary to build an ``Event`` with the
``EventBuilder`` class:

.. sourcecode:: java

    import com.getsentry.raven.Raven;
    import com.getsentry.raven.RavenFactory;
    import com.getsentry.raven.event.Event;
    import com.getsentry.raven.event.EventBuilder;
    import com.getsentry.raven.event.interfaces.ExceptionInterface;
    import com.getsentry.raven.event.interfaces.MessageInterface;

    public class MyClass {
        private static Raven raven;

        public static void main(String... args) {
            // Creation of the client with a specific DSN
            String dsn = args[0];
            raven = RavenFactory.ravenInstance(dsn);

            // It is also possible to use the DSN detection system like this
            raven = RavenFactory.ravenInstance();

            // Advanced: To specify the ravenFactory used
            raven = RavenFactory.ravenInstance(new Dsn(dsn), "com.getsentry.raven.DefaultRavenFactory");
        }

        void logSimpleMessage() {
            // This adds a simple message to the logs
            EventBuilder eventBuilder = new EventBuilder()
                            .withMessage("This is a test")
                            .withLevel(Event.Level.INFO)
                            .withLogger(MyClass.class.getName());
            raven.runBuilderHelpers(eventBuilder); // Optional
            raven.sendEvent(eventBuilder.build());
        }

        void logException() {
            try {
                unsafeMethod();
            } catch (Exception e) {
                // This adds an exception to the logs
                EventBuilder eventBuilder = new EventBuilder()
                                .withMessage("Exception caught")
                                .withLevel(Event.Level.ERROR)
                                .withLogger(MyClass.class.getName())
                                .withSentryInterface(new ExceptionInterface(e));
                raven.runBuilderHelpers(eventBuilder); // Optional
                raven.sendEvent(eventBuilder.build());
            }
        }

        void unsafeMethod() {
            throw new UnsupportedOperationException("You shouldn't call that");
        }
    }
