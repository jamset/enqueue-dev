# AMQP transport

Implements [AMQP specifications](https://www.rabbitmq.com/specification.html).
Build on top of [php amqp extension](https://github.com/pdezwart/php-amqp).

* [Installation](#installation)
* [Create context](#create-context)
* [Declare topic](#declare-topic)
* [Declare queue](#decalre-queue)
* [Bind queue to topic](#bind-queue-to-topic)
* [Send message to topic](#send-message-to-topic)
* [Send message to queue](#send-message-to-queue)
* [Consume message](#consume-message)
* [Purge queue messages](#purge-queue-messages)

## Installation

```bash
$ composer require enqueue/amqp-ext
```

## Create context

```php
<?php
use Enqueue\AmqpExt\AmqpConnectionFactory;

// connects to localhost
$connectionFactory = new AmqpConnectionFactory();

// same as above
$connectionFactory = new AmqpConnectionFactory('amqp://');

// same as above
$connectionFactory = new AmqpConnectionFactory([]);

// connect to AMQP broker at example.com
$connectionFactory = new AmqpConnectionFactory([
    'host' => 'example.com',
    'port' => 1000,
    'vhost' => '/',
    'user' => 'user',
    'pass' => 'pass',
    'persisted' => false,
]);

// same as above but given as DSN string
$connectionFactory = new AmqpConnectionFactory('amqp://user:pass@example.com:10000/%2f');

$psrContext = $connectionFactory->createContext();
```

## Declare topic.

Declare topic operation creates a topic on a broker side. 
 
```php
<?php
/** @var \Enqueue\AmqpExt\AmqpContext $psrContext */

$fooTopic = $psrContext->createTopic('foo');
$fooTopic->addFlag(AMQP_EX_TYPE_FANOUT);
$psrContext->declareTopic($fooTopic);

// to remove topic use delete topic method
//$psrContext->deleteTopic($fooTopic);
```

## Declare queue.

Declare queue operation creates a queue on a broker side. 
 
```php
<?php
/** @var \Enqueue\AmqpExt\AmqpContext $psrContext */

$fooQueue = $psrContext->createQueue('foo');
$fooQueue->addFlag(AMQP_DURABLE);
$psrContext->declareQueue($fooQueue);

// to remove topic use delete queue method
//$psrContext->deleteQueue($fooQueue);
```

## Bind queue to topic

Connects a queue to the topic. So messages from that topic comes to the queue and could be processed. 

```php
<?php
/** @var \Enqueue\AmqpExt\AmqpContext $psrContext */
/** @var \Enqueue\AmqpExt\AmqpQueue $fooQueue */
/** @var \Enqueue\AmqpExt\AmqpTopic $fooTopic */

$psrContext->bind($fooTopic, $fooQueue);
```

## Send message to topic 

```php
<?php
/** @var \Enqueue\AmqpExt\AmqpContext $psrContext */
/** @var \Enqueue\AmqpExt\AmqpTopic $fooTopic */

$message = $psrContext->createMessage('Hello world!');

$psrContext->createProducer()->send($fooTopic, $message);
```

## Send message to queue 

```php
<?php
/** @var \Enqueue\AmqpExt\AmqpContext $psrContext */
/** @var \Enqueue\AmqpExt\AmqpQueue $fooQueue */

$message = $psrContext->createMessage('Hello world!');

$psrContext->createProducer()->send($fooQueue, $message);
```

## Consume message:

```php
<?php
/** @var \Enqueue\AmqpExt\AmqpContext $psrContext */
/** @var \Enqueue\AmqpExt\AmqpQueue $fooQueue */

$consumer = $psrContext->createConsumer($fooQueue);

$message = $consumer->receive();

// process a message

$consumer->acknowledge($message);
// $consumer->reject($message);
```

## Purge queue messages:

```php
<?php
/** @var \Enqueue\AmqpExt\AmqpContext $psrContext */
/** @var \Enqueue\AmqpExt\AmqpQueue $fooQueue */

$queue = $psrContext->createQueue('aQueue');

$psrContext->purge($queue);
```

[back to index](../index.md)