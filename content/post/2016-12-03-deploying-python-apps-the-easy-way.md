---
slug: deploying-python-apps-the-easy-way
date: 2016-12-03T15:22:59.032027+00:00
draft: false
title: Deploying Python Apps The Easy Way

---

This post is heavily inspired by a presentation I saw at the
last [Paris.py meetup](https://www.meetup.com/Paris-py-Python-Django-friends/).

The speaker (Nicolas Mussat, CTO at [MeilleursAgent.com](http://meilleursagents.com)),
was kind enough to allow me to publish this on my blog.

You can find the original slides (in French) [on slideshare](
http://www.slideshare.net/diffuzed/python-application-packaging-meilleursagents).

<!--more-->

## The  problem

Let's say you are building a [flask](http://flask.pocoo.org) application
powered by [uwsgi](https://uwsgi-docs.readthedocs.io/en/latest/) and you
want to deploy it to your servers.

Here, I'm going to assume you are not using a service like
[heroku](https://www.heroku.com/) or any kind of [Platform-as-a-Service](
https://en.wikipedia.org/wiki/Platform_as_a_service).

There are plenty of cases when using a
<abbr title="Platform-as-a-service">PaaS</abbr> is a good idea, but let's assume
that for various reasons you'd like to use your own server.


## The really easy way (too easy :P)

You could be tempted to use something like:

```text
# in requirements.txt

flask
path.py
```

```console
# on the server:
$ cd my-app
$ git pull
$ pip install -r requirements.txt
$ systemctl restart uwsgi
```

This has several problems:

* What about when ``flask`` gets updated? You may end up with a version that is no
  longer compatible with the rest of your source code.

* The same issue may occur if one of the dependencies of flask gets updated in a
  non-backward compatible way.

* It's complicated to rollback to a previous version.

* Bad things will happen if github or pypi is down.

* How do you make sure all tests have been run and that they pass?

There has to be a better way!

## Separate build and deployment

A good idea would be to separate building and testing the application and its
deployment. [^1]

If you think about it, you can separate the files used by your code in two
parts:

* The *runtime*, that is the Python interpreter itself, the `uwsgi` service, and
  so on ..

* The *application*, which is all the code source and all its dependencies.
  (Which could be written in Python or be compiled `C` extensions).

If you build your package on the same linux distribution as your server, then you can
assume that all the files of the *application* you build on one machine will run
on the other.

In the worst case scenario, you'll build a `postgresql` `C` extension, which
will crash if the `postgresql` libraries are not installed, but this is easy to
fix and should not happen too often. [^2]

Of course, you could imagine using some container *ala* [docker](
https://www.docker.com/), but this goes beyond the scope of this article.
[^3]

So, to build the application package, we have to:

* Handle the versions of all the dependencies, recursively
* Bundle all the dependencies with the rest of the code.

## Building the package

### Handle dependencies

This can be done with three different ``requirements.txt`` files:

* `requirements-dev.txt`: Used by developers and continuous integration
  servers to run the tests, the linters, build the documentation and what not.

* `requirements-app.txt`: the dependencies of the application itself

* `requirements.txt`: the list of the "frozen" dependencies, generated by
  something like:

```console
$ virtualenv ~/my-app
$ source ~/my-app/bin/activate
$ pip install -r requirements-app.txt
$ pip freeze > requirements.txt
```

By the way, if you find this really close to what Ruby does with the
`Gemfile` and the `Gemfile.lock` files, and you think `pip` should better
support this kind of workflow, you
[are](https://www.kennethreitz.org/essays/a-better-pip-workflow)
[not](https://github.com/pypa/pipfile)
[alone](https://github.com/nvie/pip-tools).


If you are concerned about security here, you could:

* Use pip's new [hash-checking mode](
https://pip.pypa.io/en/stable/reference/pip_install/#hash-checking-mode).
* Create a local `pypi` index. (Lots of solutions, such as
  [devpy-server](https://pypi.python.org/pypi/devpi-server),
  [pip2pi](https://github.com/wolever/pip2pi), or even
  [bandersnatch](https://pypi.python.org/pypi/bandersnatch))

### Using a target directory for pip

The first trick is to tell `pip` to install the dependencies not in a
virtualenv or in your system, but rather in a special `vendors` directory:

```console
$ pip install --target vendors -r requirements.txt
.
├── app.py
├── requirements.txt
└── vendors
    ├── click
    ├── click-6.6.dist-info
    ├── flask
    ├── Flask-0.11.1.dist-info
    ├── itsdangerous-0.24.dist-info
    ├── itsdangerous.py
    ├── jinja2
    ├── Jinja2-2.8.dist-info
    ├── markupsafe
    ├── MarkupSafe-0.23.dist-info
    ├── werkzeug
    └── Werkzeug-0.11.11.dist-info
```


### Configuring the interpreter

The second trick is to use the [site](https://docs.python.org/3/library/site.html)
module and call `addsitedir`:

```python
import site
site.addsitedir('vendors')

from flask import Flask

app = Flask(__name__)

@app.route('/')
def index():
    return '\o/'
```

You could get away with setting `PYTHONPATH` or using `sys.path.insert(0,
'vendors')`, but using `addsitedir` will make sure Python sees the `.pth` files,
which may be required by some modules to work.

### Using setup.py to build the archive

To build the package, no need to re-invent the wheel, you can use `setup.py`
directly:

```python
# setup.py

from setuptools import setup

setup(
  name="myapp",
  version="0.1",
  py_modules=["app"],
  ...
)
```

```text
# MANIFEST.in

include myapp.conf
...
graft vendors

```

Here we make sure the configuration file for the application and the whole `vendors`
directory are included in the final package. [^4]

With this in place, you can now build the package like this:

```console
$ python setup.py sdist
$ ls dist/
myapp-0.1.tar.gz
```

## Deployment pipeline

Now that we know how to generate the package, here's an example on how we can
implement continuous delivery:

* **Build**
  * A developer pushes a tag on the git repository
  * A Jenkins job is triggered, and the tests are run
  * If they pass, a package is made and uploaded to a local storage

* **Deployment**
  * The latest package is fetched from the storage
  * The whole archive gets uploaded to the server
  * The ``uwsgi`` service is restarted on the server

The deployment phase can be managed by a tool like
[fabric](http://www.fabfile.org/) or [ansible](https://www.ansible.com/), but
that's an exercise left to the reader :)


[^1]: For more about this, feel free to read [Release Management Done Right]([http://thedailywtf.com/articles/Release-Management-Done-Right), by Alex Papdimoulis

[^2]: Using a tool like [ansible](https://www.ansible.com/) might help.

[^3]: IMHO, using docker for simple Python applications is overkill anyway, and [may not work in production as well as you'd think](https://thehftguy.com/2016/11/01/docker-in-production-an-history-of-failure/)

[^4]: `graft` is one of the many [available commands](https://docs.python.org/3/distutils/commandref.html#sdist-cmd) in a `MANIFEST.in` file.