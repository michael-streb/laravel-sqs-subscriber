# Usage

1. Add the library into your composer' dependencies
```
composer require mamitech/laravel-sqs-subscriber
```

2. Then, publish the configuration:
```
php artisan vendor:publish --provider=Mamitech\LaravelSqsSubscriber\ServiceProvider
```
at this point you should see a new file named `sqs-topic-map.php` in your `config/` directory.

3. Add these env config in you `.env` file
```
AWS_DEFAULT_REGION=ap-southeast-1
AWS_ACCESS_KEY_ID=your-public-key
AWS_SECRET_ACCESS_KEY=your-secret-key
AWS_SQS_PREFIX=https://sqs.ap-southeast-1.amazonaws.com/your-account-id
AWS_SQS_MAX_RECEIVE_MESSAGE=1
AWS_SQS_DISTRIBUTED_DEFAULT_QUEUE=test-local
AWS_SQS_DISTRIBUTED_USE_TOPIC=true
AWS_SQS_DISTRIBUTED_TOPIC_NAME=topic
AWS_SQS_DISTRIBUTED_BODY_NAME=message
```

## Troubleshoot

- If you experience a weirdness that somehow don't see this new file after publishing the config, please re-run `vendor:publish` without parameter:
```
php artisan vendor:publish
```
and select the number with `Provider: Mamitech\LaravelSqsSubscriber\ServiceProvider`. Now you should have `sqs-topic-map.php` in your config. (see [link](https://codeburst.io/if-vendor-publish-doesnt-work-laravel-ca889198f828))

- If you use Lumen, publishing config will fail due to no Laravel-specific helper method, please add new method `config_path()` as helper in your project. (see [link](https://gist.github.com/mabasic/21d13eab12462e596120))

# Configuration

In mamikos we have a standardization about how a message should be formatted to be put into queue, read:
[message format](https://mamikos.atlassian.net/wiki/spaces/MAMIKOS/pages/2056126470/Publish-subscribe+communication+across+services+in+Mamikos+System#Message-Format). By looking into the 'topic' field, we can see what is the message about.

## Define the Mapping Between Topic and a Class in Your Code

In `config/sqs-topic-map.php`, specify the mapping between topics to classes that will handle the message.
Your class should have a `handle` method receiving one `message` parameter. `message` would contains either
string or array depending on how the string inside the queue is being encoded. If it's a proper json encoded
string then it would be an array, otherwise it would be string. For details see example below.

## Add a New Connector Using `sqs-distributed` Driver into Your `config/queue.php`

Now you will need a new queue connection using the new driver called `sqs-distributed` that is provided by
this library. Please note that we are using `sqs-distributed` terms for the new driver here to get a sense
that the message is published and consumed by different services instead of how Laravel original queue
works - Laravel original queue can only works if the publisher and subscriber comes from the same service.

See example on the following section.

# Example

Suppose that you have a queue in Amazon SQS named `user-registration`. An example of the message inside the queue
looks like this:
```
{
  "topic": "user-verified",
  "message": {
    "user": {
      "email": "walker@gmail.com"
    }
  }
}
```
Here's what you need to do.

## Add New Connection in Your `config/queue.php`

Add the new connection inside your queue config using `sqs-distributed` as driver value:
```
<?php
# config/queue.php

return [
    'default' => env('QUEUE_CONNECTION', 'sync'),

    'connections' => [
        .. other connections ..

        'sqs-distributed' => [
            'driver' => 'sqs-distributed', # NOTE THIS PART
            'key' => env('AWS_ACCESS_KEY_ID', 'your-public-key'),
            'secret' => env('AWS_SECRET_ACCESS_KEY', 'your-secret-key'),
            'prefix' => env('AWS_SQS_PREFIX', 'https://sqs.us-east-1.amazonaws.com/your-account-id'),
            'queue' => env('AWS_SQS_DISTRIBUTED_DEFAULT_QUEUE', 'user-registration'),
            'topic_name' => env('AWS_SQS_DISTRIBUTED_TOPIC_NAME', 'topic'),
            'body_name' => env('AWS_SQS_DISTRIBUTED_BODY_NAME', 'message'),
            'region' => env('AWS_DEFAULT_REGION', 'us-east-1'),
        ],

        ...
```
note that in the new connection you will use `sqs-distributed` as driver value. To keep the consistency
we also name the connection as `sqs-distributed` as the name of it's driver. You can also
specify the default queue inside `queue` key that will be used when you don't specify particular
queue to listen to.

## Map Topic to Handler Class in `config/sqs-topic-map.php`

Then you should specify which class would handle each topic in your `config/sqs-topic-map.php`:
```
return [
    'user-registration => [
        'user-verified' => `App\Worker\UserVerifiedListener`
    ]
];
```
you must have the corresponding class inside `app\Worker\UserVerifiedListener.php`, this class must
have a `handle` method.

## Make Sure the Handler Class has method 'handle()' to Receive the Message.
```
<?php
namespace App\Worker;

class UserVerifiedListener
{
    public function handle($message) # THIS METHOD MUST EXISTS
    {
        $user = $message['user'];
        $email = $user['email'];
        .. your logic here ..
    }
}
```
Everytime there is a new message in the `user-registration` queue with topic of `user-verified`,
laravel will spawn a new instance of `App\Worker\UserVerifiedListener` and call method `handle`
of it by passing a `$message` parameter. This `$message` can be of any type from int, float, string
or array. It depends on the value inside `message` key in your queue message.

In the case of `$message` containing an array as in example above, then you can
directly access array data inside it using `$message['user']`. This is because when passing the parameter
to `handle`, the library will first decode the message using `json_decode`.

## Run the Worker

Now it's time to run the worker to consume and handle messages inside the queue. Because this
library utilize laravel' original queue handler, we can simply run the queue worker as usual
by specifying which connection we want to run the worker for.

In our case, the connection would be `sqs-distributed`, so the command to run the worker is:
```
php artisan queue:work sqs-distributed
```
The command above will run the worker to listen to sqs-distributed with default queue. If you
want to specify specific queue or even multiple queue to listen to, you can add `--queue`
parameter as described in [Laravel doc](https://laravel.com/docs/5.7/queues#running-the-queue-worker)

That's it! now your worker will consume the message from Amazon SQS. Just make sure that your worker
has a supervisor program such as systemd to run it so that when something unexpectedly happened and
crash the worker, it will automatically be restarted.

# Advance Features

## Receive Multiple Messages at Once

In the newest version, this library support to receive multiple messages at once, just add this new key in `.env` and `config/queue.php` files
```
// .env
AWS_SQS_MAX_RECEIVE_MESSAGE=1 # max number is 10

// config/queue.php
'connections' => [
    .. other connections ..

    'sqs-distributed' => [
        .. other keys ..
        'max_receive_message' => (int) env('AWS_SQS_MAX_RECEIVE_MESSAGE', 1),
    ],
]
```
