# Server Requirement
- Redis `sudo apt install redis-server`
- Composer [here](https://getcomposer.org/download/)
- npm `sudo apt install nodejs npm`
- laravel-echo-server `npm install -g laravel-echo-server`
- then answer a few questions, and answer as below
    - Do you want to run this server in development mode? `Yes`
    - Which port would you like to serve from? `6001`
    - Which database would you like to use to store presence channel members? `redis`
    - Enter the host of your Laravel authentication server. `http://localhost`
    - Will you be serving on http or https? `https`
    - (after that you will be asked to enter the patch where your SSL certificate file and SSL server key file are stored)
    - Do you want to generate a client ID/Key for HTTP API? `Yes`
    - Do you want to setup cross domain access to the API? `Yes`
    - Specify the URI that may access the API: `http://localhost:80`
    - Enter the HTTP methods that are allowed for CORS: `GET, POST`
    - Enter the HTTP headers that are allowed for CORS: `Origin, Content-Type, X-Auth-Token, X-Requested-With, Accept, Authorization, X-CSRF-TOKEN, X-Socket-Id`
    - What do you want this config to be saved as? `laravel-echo-server.json`
    

# Laravel Dependencies Requirement
- predis
- laravel-echo
- socket.io-client ^2.4.0 [issue](https://github.com/laravel/echo/issues/237#issuecomment-731308117)

# Installation
- Install fresh laravel via Composer `laravel new laravel-socket-io`
- Go to Laravel directory `cd laravel-socket-io`
- Add predis `composer require predis/predis`
- Add laravel-echo and socket.io client `npm install --save laravel-echo socket.io-client@2.4.0`
- Create laravel-echo-server configuration `laravel-echo-server init`
- Specify `.env`
```
...
# Broadcast
BROADCAST_DRIVER=redis
...
# Redis
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379
REDIS_CLIENT=predis
REDIS_PREFIX=""
...
# Laravel Echo Server
LARAVEL_ECHO_SERVER_REDIS_HOST=127.0.0.1
LARAVEL_ECHO_SERVER_REDIS_PORT=6379
LARAVEL_ECHO_SERVER_REDIS_PASSWORD=null
```
- Create new file `resources/js/echo.js`
```js
# echo.js
import Echo from 'laravel-echo';

window.io = require('socket.io-client');

window.Echo = new Echo({
    broadcaster: 'socket.io',
    host: window.location.hostname + ':6001'
});
```
- Add `require('./echo.js');` at bottom of file `resources/js/app.js`
- Run `npm install`
- Run `npm run dev`
- Enable channel route, Uncomment `App\Providers\BroadcastServiceProvider::class,`
- Add route `routes/channels.php`
```php
...
Broadcast::channel('EveryoneChannel', function () {
    return true;
});
...
```
- Create new event `php artisan make:event EveryoneEvent` and edit
```php
<?php

namespace App\Events;

use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class EveryoneEvent implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;
    
    /**
    * The name of the queue connection to use when broadcasting the event.
    *
    * @var string
    */
    public $connection = 'redis';

    /**
    * The name of the queue on which to place the broadcasting job.
    *
    * @var string
    */
    public $queue = 'default';

    /**
     * Create a new event instance.
     *
     * @return void
     */
    public function __construct()
    {
        //
    }

    /**
     * Get the channels the event should broadcast on.
     *
     * @return \Illuminate\Broadcasting\Channel|array
     */
    public function broadcastOn()
    {
        return new Channel('EveryoneChannel');
    }

    /*
     * The Event's broadcast name.
     * 
     * @return string
     */
    public function broadcastAs()
    {
        return 'EveryoneMessage';
    }
    
    /*
     * Get the data to broadcast.
     *
     * @return array
     */
    public function broadcastWith()
    {
        return [
            'message'=> 'Hello!'
        ];
    }
}
```
- Add route for send and receiver broadcast at file `routes/web.php`
```php
...
Route::get('/send', function () {
    broadcast(new App\Events\EveryoneEvent());
    return response('Sent');
});

Route::get('/receiver', function () {
    return view('receiver');
});
...
```
- Create new view `resources/views/receiver.blade.php`
```html
<!DOCTYPE html>
<html lang="en">
    <head>
        <meta charset="utf-8">
        <meta name="viewport" content="width=device-width, initial-scale=1">

        <title>Receiver</title>

        <script type="application/javascript" src="{{ asset('js/app.js') }}"></script>
    </head>
    <body>
        <div id="messages"></div>
        <script
            src="https://code.jquery.com/jquery-2.2.4.js"
            integrity="sha256-iT6Q9iMJYuQiMWNd9lDyBUStIq/8PuOW33aOqmvFpqI="
            crossorigin="anonymous"></script>
        <script type="text/javascript">
           window.Echo.channel('EveryoneChannel')
               .listen('.EveryoneMessage', function (e) {
                   $('#messages').append('<p>' + e.message + '</p>');
                })
        </script>
    </body>
</html>
```
- Start laravel-echo-server `laravel-echo-server start`
- Start Laravel queue `php artisan queue:listen redis --queue=default`
- Start Laravel app `php artisan serve`
- Open URL two tabs/browsers
  1. http://localhost:8000/send
  2. http://localhost:8000/receiver
- When you hit first URL you will see `Hello!` from second URL.
- DONE, have a good day!
