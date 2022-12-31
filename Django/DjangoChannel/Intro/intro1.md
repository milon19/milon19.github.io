# Django Channel - Intro #1

I am following [Channels official](https://channels.readthedocs.io/en/stable/tutorial/index.html) doc for this part.

## Creating a django project.

Make a folder named `djangochannels` and also make a virtual environment and activate it.

- Follow the following script to do it.

```shell script
$ mkdir djangochannels
$ virtualenv venv
$ source ./venv/bin/activate
```

- Install Django

```shell script
$ pip3 install django
```

- Run the following command for create a new django project.

```shell script
$ django-admin startproject djangochannels .
```

Folder structure will look like this tree.

```
├── djangochannels
│   ├── asgi.py
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── manage.py
└── venv
```

Make a `.gitignore` file and add `/venv`. In case you push your code to vcs

## Creating the Chat app

Make sure you’re in the same directory as `manage.py` and type this command:

```shell script
$ python3 manage.py startapp chat
```

That’ll create a directory chat, which is laid out like this:

```
├── chat
│   ├── admin.py
│   ├── apps.py
│   ├── __init__.py
│   ├── migrations
│   │   └── __init__.py
│   ├── models.py
│   ├── tests.py
│   └── views.py
├── djangochannels
│   ├── asgi.py
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
└── manage.py
```

For the purposes of this tutorial, we will only be working with `chat/views.py` and `chat/__init__.py`. So remove all other files from the chat directory. <b>[I am keeping `apps.py` also]</b>

After removing unnecessary files, the chat directory should look like:

```
.
├── chat
│   ├── apps.py             # kept this
│   ├── __init__.py
│   ├── migrations
│   │   └── __init__.py
│   └── views.py
├── djangochannels
│   ├── asgi.py
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
└── manage.py
```

Edit `djangochannel/settings.py` and add `"chat.apps.ChatConfig"` to `INSTALLED_APPS`. It'll look like this:

```python
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",

    "chat.apps.ChatConfig",       # Added here
]
```

## Create Index view

Lets create the first view, an index view that lets you type the name of a chat room to join.

Create a `templates` directory in your `chat` directory. Within the `templates` directory you have just created, create another directory called `chat`, and within that create a file called `index.html` to hold the template for the index view.

Your `chat` directory should now look like:

```
chat
├── apps.py
├── __init__.py
├── migrations
│   └── __init__.py
├── templates
│   └── chat
│       └── index.html
└── views.py
```

Put the following code in `chat/templates/chat/index.html`:

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8" />
    <title>Chat Rooms</title>
  </head>
  <body>
    What chat room would you like to enter?<br />
    <input id="room-name-input" type="text" size="100" /><br />
    <input id="room-name-submit" type="button" value="Enter" />

    <script>
      document.querySelector("#room-name-input").focus();
      document.querySelector("#room-name-input").onkeyup = function (e) {
        if (e.keyCode === 13) {
          // enter, return
          document.querySelector("#room-name-submit").click();
        }
      };

      document.querySelector("#room-name-submit").onclick = function (e) {
        var roomName = document.querySelector("#room-name-input").value;
        window.location.pathname = "/chat/" + roomName + "/";
      };
    </script>
  </body>
</html>
```

Create the view function for the room view. Put the following code in `chat/views.py`:

```python
from django.shortcuts import render


def index(request):
    return render(request, "chat/index.html")
```

To call the view, we need to map it to a URL - and for this we need a URLconf.

To create a URLconf in the `chat` directory, create a file called `urls.py` and include the following code:

```python
from django.urls import path

from chat import views

app_name = "chat"

urlpatterns = [
    path("", views.index, name="index"),
]
```

The next step is to point the root URLconf at the `chat.urls` module. In `djangochannels/urls.py`, add an import for `django.urls.include` and insert an `include()` in the `urlpatterns` list, so you have:

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path("admin/", admin.site.urls),
    path("chat/", include("chat.urls", namespace="chat"))
]
```

Let’s verify that the index view works. Run the following command:

```shell script
$ python3 manage.py runserver
```

Go to http://127.0.0.1:8000/chat/ in your browser and you should see the text “What chat room would you like to enter?” along with a text input to provide a room name.

Type in “lobby” as the room name and press enter. You should be redirected to the room view at http://127.0.0.1:8000/chat/lobby/ but we haven’t written the room view yet, so you’ll get a “Page not found” error page.

## Integrate the Channels library

So far we’ve just created a regular Django app; we haven’t used the Channels library at all. Now it’s time to integrate Channels.

For install channels [see more here](https://channels.readthedocs.io/en/stable/installation.html)

To install it run:

```shell script
$ pip3 install -U channels["daphne"]
```

This will install the Daphne’s ASGI version of the runserver management command.

Let’s start by creating a [routing configuration](https://channels.readthedocs.io/en/stable/topics/routing.html) for Channels. A Channels routing configuration is an ASGI application that is similar to a Django URLconf, in that it tells Channels what code to run when an HTTP request is received by the Channels server.

Start by adjusting the `djangochannels/asgi.py` file to include the following code:

```python
import os

from django.core.asgi import get_asgi_application

from channels.routing import ProtocolTypeRouter

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "djangochannels.settings")

application = ProtocolTypeRouter(
    {
        "http": get_asgi_application(),
        # Just HTTP for now. (We can add other protocols later.)
    }
)
```

Now add the Daphne library to the list of installed apps, in order to enable an ASGI versions of the runserver command.

Edit the ``djangochannels/settings.py` file and add 'daphne' to the top of the INSTALLED_APPS setting. It’ll look like this:

```python
INSTALLED_APPS = [
    'daphne',

    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",

    "chat.apps.ChatConfig",
]
```

> The Daphne development server will conflict with any other third-party apps that require an overloaded or replacement runserver command. In order to solve such issues, make sure daphne is at the top of your INSTALLED_APPS, or remove the offending app altogether. [see more](https://channels.readthedocs.io/en/stable/tutorial/part_1.html#integrate-the-channels-library)

You’ll also need to point Daphne at the root routing configuration. Add the following code in `djangochannels/settings.py` file

```python
ASGI_APPLICATION = "djangochannels.asgi.application"
```

Let’s ensure that the Channels development server is working correctly. Run the following command:

```shell script
$ python3 manage.py runserver
```

Notice the line beginning with `Starting ASGI/Daphne …`. This indicates that the Daphne development server has taken over from the Django development server.

Go to http://127.0.0.1:8000/chat/ in your browser and you should still see the index page that we created before.

This tutorial continues in [Part #2](./intro2.md).
