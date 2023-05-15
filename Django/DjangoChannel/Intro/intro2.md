# Django Channel - Intro #2 - Implement a Chat Server

## Add the room view

Create a new file `chat/templates/chat/room.html`. Your app directory should now look like:

```
chat
├── apps.py
├── templates
│   └── chat
│       ├── index.html
│       └── room.html
├── urls.py
└── views.py
```

Create a view template for room view in `room.html`:

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8" />
    <title>Chat Room</title>
  </head>
  <body>
    <textarea id="chat-log" cols="100" rows="20"></textarea><br />
    <input id="chat-message-input" type="text" size="100" /><br />
    <input id="chat-message-submit" type="button" value="Send" />
    {{ room_name|json_script:"room-name" }}
    <script>
      const roomName = JSON.parse(
        document.getElementById("room-name").textContent
      );

      const chatSocket = new WebSocket(
        "ws://" + window.location.host + "/ws/chat/" + roomName + "/"
      );

      chatSocket.onmessage = function (e) {
        const data = JSON.parse(e.data);
        document.querySelector("#chat-log").value += data.message + "\n";
      };

      chatSocket.onclose = function (e) {
        console.error("Chat socket closed unexpectedly");
      };

      document.querySelector("#chat-message-input").focus();
      document.querySelector("#chat-message-input").onkeyup = function (e) {
        if (e.keyCode === 13) {
          // enter, return
          document.querySelector("#chat-message-submit").click();
        }
      };

      document.querySelector("#chat-message-submit").onclick = function (e) {
        const messageInputDom = document.querySelector("#chat-message-input");
        const message = messageInputDom.value;
        chatSocket.send(
          JSON.stringify({
            message: message,
          })
        );
        messageInputDom.value = "";
      };
    </script>
  </body>
</html>
```

Create a view function for the room view in `chat/view.py`:

```python
from django.shortcuts import render


def index(request):
    return render(request, "chat/index.html")


def room(request, room_name):
    return render(request, "chat/room.html", {"room_name": room_name})
```

Also create a router for the room view in `chat/urls.py`:

```python
from django.urls import path

from chat import views

app_name = "chat"

urlpatterns = [
    path("", views.index, name="index"),
    path("<str:room_name>/", views.room, name="room"),
]
```

Now start the development server:

```shell
python3 manage.py runserver
```

Go to [http://127.0.0.1:8000/chat/](http://127.0.0.1:8000/chat/) in your browser and to see the index page.

Type in `lobby` as the room name and press enter. You should be redirected to the room page at [http://127.0.0.1:8000/chat/lobby/](http://127.0.0.1:8000/chat/lobby/) which now displays an empty chat log.

Type the message “hello” and press enter. Nothing happens. In particular the message does not appear in the chat log. Why?

The room view is trying to open a WebSocket to the URL [ws://127.0.0.1:8000/ws/chat/lobby/](ws://127.0.0.1:8000/ws/chat/lobby/) but we haven’t created a consumer that accepts WebSocket connections yet. If you open your browser’s JavaScript console, you should see an error that looks like:

> WebSocket connection to `ws://127.0.0.1:8000/ws/chat/lobby/` failed:

## Write a first consumer

When Django accepts an HTTP request, it consults the root URLconf to lookup a view function, and then calls the view function to handle the request. Similarly, when Channels accepts a WebSocket connection, it consults the root routing configuration to lookup a consumer, and then calls various functions on the consumer to handle events from the connection.

We will write a basic consumer that accepts WebSocket connections on the path `/ws/chat/ROOM_NAME/` that takes any message it receives on the WebSocket and echos it back to the same WebSocket.

Create a new file `chat/consumers.py`. Your app directory should now look like:

```
chat
├── apps.py
├── consumers.py
├── templates
│   └── chat
│       ├── index.html
│       └── room.html
├── urls.py
└── views.py
```

Put the following code in `chat/consumers.py`:

```python
import json

from channels.generic.websocket import WebsocketConsumer


class ChatConsumer(WebsocketConsumer):
    def connect(self):
        self.accept()

    def disconnect(self, close_code):
        pass

    def receive(self, text_data):
        text_data_json = json.loads(text_data)
        message = text_data_json["message"]

        self.send(text_data=json.dumps({"message": message}))
```

This is a synchronous WebSocket consumer that accepts all connections, receives messages from its client, and echos those messages back to the same client. For now it does not broadcast messages to other clients in the same room.

> Channels also supports writing asynchronous consumers for greater performance. However any asynchronous consumer must be careful to avoid directly performing blocking operations, such as accessing a Django model. See the Consumers reference for more information about writing asynchronous consumers.

We need to create a routing configuration for the `chat` app that has a route to the consumer. Create a new file `chat/routing.py`. Your app directory should now look like:

Put the following code in `chat/routing.py`:

```python
from django.urls import re_path

from chat import consumers

websocket_urlpatterns = [
    re_path(r"ws/chat/(?P<room_name>\w+)/$", consumers.ChatConsumer.as_asgi()),
]
```

We call the `as_asgi()` classmethod in order to get an ASGI application that will instantiate an instance of our consumer for each user-connection. This is similar to Django’s `as_view()`, which plays the same role for per-request Django view instances.

(Note we use `re_path()` due to limitations in URLRouter.)

The next step is to point the main ASGI configuration at the chat.routing module. In `djangochannels/asgi.py`, import `AuthMiddlewareStack`, `URLRouter`, and `chat.routing`; and insert a `websocket` key in the `ProtocolTypeRouter` list in the following format:

```python
import os

from django.core.asgi import get_asgi_application

from channels.auth import AuthMiddlewareStack
from channels.security.websocket import AllowedHostsOriginValidator
from channels.routing import ProtocolTypeRouter, URLRouter

from chat import routing

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "djangochannels.settings")

application = ProtocolTypeRouter(
    {
        "http": get_asgi_application(),
        "websocket": AllowedHostsOriginValidator(
            AuthMiddlewareStack(URLRouter(routing.websocket_urlpatterns))
        ),
    }
)
```

This root routing configuration specifies that when a connection is made to the Channels development server, the `ProtocolTypeRouter` will first inspect the type of connection. If it is a WebSocket connection (<b>ws://</b> or <b>wss://</b>), the connection will be given to the `AuthMiddlewareStack`.

The `AuthMiddlewareStack` will populate the connection’s <b>scope</b> with a reference to the currently authenticated user, similar to how Django’s `AuthenticationMiddleware `populates the request object of a view function with the currently authenticated user. (Scopes will be discussed later in this tutorial.) Then the connection will be given to the `URLRouter`.

The `URLRouter` will examine the HTTP path of the connection to route it to a particular consumer, based on the provided url patterns.

Let’s verify that the consumer for the `/ws/chat/ROOM_NAME/` path works. Run migrations to apply database changes (Django’s session framework needs the database) and then start the Channels development server and Go to the room page at http://127.0.0.1:8000/chat/lobby/ which now displays an empty chat log.

Type the message “hello” and press enter. You should now see “hello” echoed in the chat log.

However if you open a second browser tab to the same room page at http://127.0.0.1:8000/chat/lobby/ and type in a message, the message will not appear in the first tab. For that to work, we need to have multiple instances of the same ChatConsumer be able to talk to each other. Channels provides a channel layer abstraction that enables this kind of communication between consumers.

## Optional

You can also verify your connection using https://websocketking.com/. For that you need allow the origin.

Edit `ALLOWED_HOSTS` variable in `djangochannels/setting.py` like follows:

```python
ALLOWED_HOSTS = ["*"]
```

Put the ws://127.0.0.1:8000/ws/chat/lobby/ and click the connect button.

You can see the following messages in Output section in https://websocketking.com/.

```
Connected to ws://127.0.0.1:8000/ws/chat/lobby/
Connecting to ws://127.0.0.1:8000/ws/chat/lobby/
```

Type the following message in payload and press Send button.

```json
{
  "message": "hello"
}
```

You will get the same message as output.

## Enable a channel layer

A channel layer is a kind of communication system. It allows multiple consumer instances to talk with each other, and with other parts of Django.

A channel layer provides the following abstractions:

- A channel is a mailbox where messages can be sent to. Each channel has a name. Anyone who has the name of a channel can send a message to the channel.
- A group is a group of related channels. A group has a name. Anyone who has the name of a group can add/remove a channel to the group by name and send a message to all channels in the group. It is not possible to enumerate what channels are in a particular group.

Every consumer instance has an automatically generated unique channel name, and so can be communicated with via a channel layer.

In our chat application we want to have multiple instances of ChatConsumer in the same room communicate with each other. To do that we will have each `ChatConsumer` add its channel to a group whose name is based on the room name. That will allow `ChatConsumers` to transmit messages to all other `ChatConsumers` in the same room.

We will use a channel layer that uses Redis as its backing store. To start a Redis server on port `6379`, run the following command:

Using docker:

```shell
$ docker run -p 6379:6379 -d redis:5
```

You can also install redis in locally by following this. [Install Redis.](https://redis.io/docs/getting-started/)

We need to install `channels_redis` so that Channels knows how to interface with Redis. Run the following command:

```shell
$ pip3 install channels_redis
```

Before we can use a channel layer, we must configure it. Edit the `djangochannels/settings.py` file and add a `CHANNEL_LAYERS` setting to the bottom. It should look like:

```python
CHANNEL_LAYERS = {
    "default": {
        "BACKEND": "channels_redis.core.RedisChannelLayer",
        "CONFIG": {
            "hosts": [("127.0.0.1", 6379)],
        },
    },
}
```

> It is possible to have multiple channel layers configured. However most projects will just use a single 'default' channel layer.

Let’s make sure that the channel layer can communicate with Redis. Open a Django shell and run the following commands:

```shell
$ python3 manage.py shell
>>> import channels.layers
>>> channel_layer = channels.layers.get_channel_layer()
>>> from asgiref.sync import async_to_sync
>>> async_to_sync(channel_layer.send)('test_channel', {'type': 'hello'})
>>> async_to_sync(channel_layer.receive)('test_channel')
{'type': 'hello'}
```
