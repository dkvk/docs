Implementing an event bus
=========================

The classes and interfaces from this package can also be used to set up
an event bus. The characteristics of an event bus are:

-  It handles *events*, i.e. informational messages
-  Zero or more *event subscribers* will be notified of the occurance of
   an event
-  The behavior of the event bus is extensible: *middlewares* are
   allowed to do things before or after handling an event

Setting up the event bus
------------------------

At least we need an instance of ``MessageBusSupportingMiddleware``:

.. code-block::  php

    use SimpleBus\Message\Bus\Middleware\MessageBusSupportingMiddleware;

    $eventBus = new MessageBusSupportingMiddleware();

Finish handling an event, before handling the next
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

We want to make sure that events are always fully handled before other
events will be handled, so we add a specialized middleware for that:

.. code-block::  php

    use SimpleBus\Message\Bus\Middleware\FinishesHandlingMessageBeforeHandlingNext;

    $eventBus->appendMiddleware(new FinishesHandlingMessageBeforeHandlingNext());

Defining the event subscriber collection
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

We want any number of event subscribers to be notified of a given event.
We first need to define the collection of event subscribers that are
available in the application. We should make this *event subscriber
collection* lazy-loading, or every event subscriber will be fully
loaded, even though it is not going to be used:

.. code-block::  php

    use SimpleBus\Message\CallableResolver\CallableCollection;
    use SimpleBus\Message\CallableResolver\ServiceLocatorAwareCallableResolver;

    // Provide a map of event names to callables. You can provide actual callables, or lazy-loading ones.
    $eventSubscribersByEventName = [
        Fully\Qualified\Class\Name\Of\Event::class => [ // an array of "callables",
            ...,
            ...
        ]
        ...
    ];

Each of the provided "callables" can be one of the following things:

-  An actual `PHP
   callable <http://php.net/manual/en/language.types.callable.php>`__,
-  A service id (string) which the service locator (see below) can
   resolve to a PHP callable,
-  An array of which the first value is a service id (string), which the
   service locator can resolve to a regular object, and the second value
   is a method name.

For backwards compatibility an object with a ``notify()`` method also
counts as a "callable".

.. code-block::  php

    // Provide a service locator callable. It will be used to instantiate a subscriber service whenever requested.
    $serviceLocator = function ($serviceId) {
        $handler = ...;

        return $handler;
    };

    $eventSubscriberCollection = new CallableCollection(
        $eventSubscribersByEventName,
        new ServiceLocatorAwareCallableResolver($serviceLocator)
    );

Resolving the event subscribers for an event
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The name of an event
^^^^^^^^^^^^^^^^^^^^

First we need a way to resolve the name of an event. You can use the
fully-qualified class name (FQCN) of an event object as its name:

.. code-block::  php

    use SimpleBus\Message\Name\ClassBasedNameResolver;

    $eventNameResolver = new ClassBasedNameResolver();

Or you can ask event objects what their name is:

.. code-block::  php

    use SimpleBus\Message\Name\NamedMessageNameResolver;

    $eventNameResolver = new NamedMessageNameResolver();

In that case your events have to implement ``NamedMessage``:

.. code-block::  php

    use SimpleBus\Message\Name\NamedMessage;

    class YourEvent implements NamedMessage
    {
        public static function messageName()
        {
            return 'your_event';
        }
    }

    .. rubric:: Implementing your own ``MessageNameResolver``
       :name: implementing-your-own-messagenameresolver

    If you want to use another rule to determine the name of an event,
    create a class that implements
    ``SimpleBus\Message\Name\MessageNameResolver``.

Resolving the event subscribers based on the name of the event
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Using the ``MessageNameResolver`` of your choice, you can now let the
*event subscribers resolver* find the right event subscribers for a
given event.

.. code-block::  php

    use SimpleBus\Message\Subscriber\Resolver\NameBasedMessageSubscriberResolver;

    $eventSubscribersResolver = new NameBasedMessageSubscriberResolver(
        $eventNameResolver,
        $eventSubscriberCollection
    );

Finally, we should add some middleware to the event bus that notifies
all of the resolved event subscribers:

.. code-block::  php

    use SimpleBus\Message\Subscriber\NotifiesMessageSubscribersMiddleware;

    $eventBus->appendMiddleware(
        new NotifiesMessageSubscribersMiddleware(
            $eventSubscribersResolver
        )
    );

Using the event bus: an example
-------------------------------

Consider the following event:

.. code-block::  php

    class UserRegistered
    {
        private $userId;

        public function __construct(UserId $userId)
        {
            $this->userId = $userId;
        }

        public function userId()
        {
            return $this->userId;
        }
    }

This event conveys the information that "a new user was registered". The
message data consists of the unique identifier of the user that was
registered. This information is required for event subscribers to act
upon the event.

A subscriber for this event looks like this:

.. code-block::  php

    class SendWelcomeMailWhenUserRegistered
    {
        ...

        public function notify($message)
        {
            $user = $this->userRepository->byId($message->userId());

            // send the welcome mail
        }
    }

We should register this subscriber as a service and add the service id
to the `event subscriber collection <#event-subscriber-collection>`__.
Since we have already fully configured the event bus, we can just start
creating a new event object and let the event bus handle it. Eventually
the event will be passed as a message to the
``SendWelcomeMailWhenUserRegistered`` event subscriber:

.. code-block::  php

    $userId = $this->userRepository->nextIdentity();

    $event = new UserRegistered($userId);

    $eventBus->handle($event);

    .. rubric:: Implementing your own event bus middleware
       :name: implementing-your-own-event-bus-middleware

    It's very easy to extend the behavior of the event bus. You can
    create a class that implements ``MessageBusMiddleware``:

    .. code-block::  php

        use SimpleBus\Message\Bus\Middleware\MessageBusMiddleware;

        /**
         * Marker interface for domain events that should be stored in the event store
         */
        interface DomainEvent
        {
        }

        class StoreDomainEvents implements MessageBusMiddleware
        {
            ...

            public function handle($message, callable $next)
            {
                if ($message instanceof DomainEvent) {
                    // store the domain event
                    $this->eventStore->add($message);
                }

                // let other middlewares do their job
                $next($message);
            }
        }

    You should add an instance of that class as middleware to any
    ``MessageBusSupportingMiddleware`` instance (like the event bus we
    created earlier):

    .. code-block::  php

        $eventBus->appendMiddleware(new StoreDomainEvents());

    Make sure that you do this at the right place, before or after you
    add the other middlewares.

    Calling ``$next($message)`` will make sure that the next middleware
    in line is able to handle the message.

    .. rubric:: Logging messages
       :name: logging-messages

    To log every message that passes through the event bus, add the
    ``LoggingMiddleware`` right before the
    ``NotifiesMessageSubscribersMiddleware``. Make sure to set up a
    `PSR-3 compliant logger <http://www.php-fig.org/psr/psr-3/>`__
    first:

    .. code-block::  php

        use Psr\Log\LoggerInterface;
        use Psr\Log\LogLevel;
        use SimpleBus\Message\Logging\LoggingMiddleware;

        // $logger is an instance of LoggerInterface
        $logger = ...;
        $loggingMiddleware = new LoggingMiddleware($logger, LogLevel::DEBUG);
        $eventBus->appendMiddleware($loggingMiddleware);

Continue to read about `recording events and handling
them <message_recorder.md>`__.
