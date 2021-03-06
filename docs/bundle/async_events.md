# Async events

The EnqueueBundle allows you to dispatch events asynchronously. 
Behind the scene it replaces your listener with one that sends a message to MQ. 
The message contains the event object. 
The consumer, once it receives the message, restores the event and dispatches it to only async listeners.

Async listeners benefits:

* The response time lesser. It has to do less work.
* Better fault tolerance. Bugs in async listener does not affect user. Messages will wait till you fix bugs.
* Better scaling. Add more consumers to meet the load.

_**Note**: The php serializer transformer (the default one) does not work on Symfony prior 3.0. The event contains eventDispatcher and therefor could not be serialized. You have to register a transformer for every async event. Read the [event transformer](#event-transformer)._

## Configuration

I suppose you already [installed the bundle](quick_tour.md#install). 
Now, you have to enable `async_events`. 
If you do not enable it, events will be processed as before: synchronously.

```yaml
# app/config/config.yml

enqueue:
   async_events:
      enabled: true
      # if you'd like to send send messages onTerminate use spool_producer (it makes response time even lesser):
      # spool_producer: true
```

## Usage

To make your listener async you have add `async: true` attribute to the tag `kernel.event_listener`, like this:

```yaml
# app/config/config.yml

services:
    acme.foo_listener:
        class: 'AcmeBundle\Listener\FooListener'
        tags:
            - { name: 'kernel.event_listener', async: true, event: 'foo', method: 'onEvent' }
```

or to `kernel.event_subscriber`:

```yaml
# app/config/config.yml

services: 
    test_async_subscriber:
        class: 'AcmeBundle\Listener\TestAsyncSubscriber'
        tags:
            - { name: 'kernel.event_subscriber', async: true }
```

That's basically it. The rest of the doc describes advanced features. 

## Advanced Usage.

You can also add an async listener directly and register a custom message processor for it:

```yaml
# app/config/config.yml

services:
    acme.async_foo_listener:
        class: 'Enqueue\Bundle\Events\AsyncListener'
        public: false
        arguments: ['@enqueue.client.producer', '@enqueue.events.registry']
        tags:
          - { name: 'kernel.event_listener', event: 'foo', method: 'onEvent' }
```

The message processor must subscribe to `event.foo` topic. The message queue topics names for event follow this patter `event.{eventName}`.

```php
<?php

use Enqueue\Bundle\Events\Registry;
use Enqueue\Client\TopicSubscriberInterface;
use Enqueue\Psr\PsrContext;
use Enqueue\Psr\PsrMessage;
use Enqueue\Psr\PsrProcessor;

class FooEventProcessor implements PsrProcessor, TopicSubscriberInterface
{
    /**
     * @var Registry
     */
    private $registry;

    /**
     * @param Registry $registry
     */
    public function __construct(Registry $registry)
    {
        $this->registry = $registry;
    }

    public function process(PsrMessage $message, PsrContext $context)
    {
        if (false == $eventName = $message->getProperty('event_name')) {
            return self::REJECT;
        }
        if (false == $transformerName = $message->getProperty('transformer_name')) {
            return self::REJECT;
        }

        // do what you want with the event.
        $event = $this->registry->getTransformer($transformerName)->toEvent($eventName, $message);
        
        
        return self::ACK;
    }

    public static function getSubscribedTopics()
    {
        return ['event.foo'];
    }
}
```


## Event transformer

The bundle uses [php serializer](https://github.com/php-enqueue/enqueue-dev/blob/master/pkg/enqueue-bundle/Events/PhpSerializerEventTransformer.php) transformer by default to pass events through MQ.
You could create a transformer for the given event type. The transformer must implement `Enqueue\Bundle\Events\EventTransformer` interface.
Consider the next example. It shows how to send an event that contains Doctrine entity as a subject  
 
```php
<?php
namespace AcmeBundle\Listener;

// src/AcmeBundle/Listener/FooEventTransformer.php

use Enqueue\Client\Message;
use Enqueue\Consumption\Result;
use Enqueue\Psr\PsrMessage;
use Enqueue\Util\JSON;
use Symfony\Component\EventDispatcher\Event;
use Enqueue\Bundle\Events\EventTransformer;
use Doctrine\Bundle\DoctrineBundle\Registry;
use Symfony\Component\EventDispatcher\GenericEvent;

class FooEventTransformer implements EventTransformer
{
    /** @var Registry @doctrine */
    private $doctrine;

    public function __construct(Registry $doctrine)
    {
        $this->doctrine = $doctrine;
    }

    /**
     * {@inheritdoc}
     * 
     * @param GenericEvent $event
     */
    public function toMessage($eventName, Event $event = null)
    {
        $entity = $event->getSubject();
        $entityClass = get_class($event);
        
        $manager = $this->doctrine->getManagerForClass($entityClass);
        $meta = $manager->getClassMetadata($entityClass);

        $id = $meta->getIdentifierValues($entity);
        
        $message = new Message();
        $message->setBody([
            'entityClass' => $entityClass, 
            'entityId' => $id,
            'arguments' => $event->getArguments()
        ]);

        return $message;
    }

    /**
     * {@inheritdoc}
     */
    public function toEvent($eventName, PsrMessage $message)
    {
        $data = JSON::decode($message->getBody());
        
        $entityClass = $data['entityClass'];
        
        $manager = $this->doctrine->getManagerForClass($entityClass);
        if (false == $entity = $manager->find($entityClass, $data['entityId'])) {
            return Result::reject('The entity could not be found.');
        }
        
        return new GenericEvent($entity, $data['arguments']);
    }
}
```

and register it:

```yaml
# app/config/config.yml

services:
    acme.foo_event_transofrmer:
        class: 'AcmeBundle\Listener\FooEventTransformer'
        arguments: ['@doctrine']
        tags:
            - {name: 'enqueue.event_transformer', eventName: 'foo' }
```

The `eventName` attribute accepts a regexp. You can do next `eventName: '/foo\..*?/'`. 
It uses this transformer for all event with the name beginning with `foo.`

[back to index](../index.md)