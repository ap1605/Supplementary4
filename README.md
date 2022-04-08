- [Introduction](#introduction)
  * [What is Docker?](#what-is-docker)
  * [How are we going to use Docker?](#how-are-we-going-to-use-docker)
  * [What does the microservice do?](#what-does-the-microservice-do)
- [Steps to integrate a change and deploy it](#steps-to-integrate-a-change-and-deploy-it)
  * [Setting up the Build Environment](#setting-up-the-build-environment)
  * [Testing](#testing)
      - [Running the Webserver and Docker Compose](#running-the-webserver-and-docker-compose)
      - [The Dockerfile](#the-dockerfile)
      - [Using Docker Compose with our Dockerfile](#using-docker-compose-with-our-dockerfile)
  * [Distributing](#distributing)
- [Automating CI/CD using Pipelines](#automating-ci-cd-using-pipelines)
  * [The `build` Stage](#the-build-stage)
  * [The `test` Stage](#the-test-stage)
  * [The `dockerize` Stage](#the-dockerize-stage)
  * [Testing the deployed image](#testing-the-deployed-image)

# Introduction

In Part 1 of the exercise, we learned how to integrate simple unit testing for a
Python package using pytest into a pipeline.  Now what if the software we want
to test is much more complicated than that Python package?

Let's say the software we wanted to test was an entire microservice running on a
web server with a database backend.  And suppose we wanted to do integration
testing and systems testing to make sure that the system
works well as a whole (precluding any mocking for the database system and other external systems).  For complex software like this, it takes a lot of care
and effort to set up the preconditions correctly before each test run.  Things
that you could easily get wrong are:

* Not having the correct web server version installed on your system.
* Having the correct web server but not not configuring it corretly.
* Not having the database server running in your system.
* Having the database running but not initialized with the correct data set.

If your dedicated test machine is used to test other software that needs
web servers of different versions configured differently with their own data sets
stored in databases, you can imagine what kind of issues you may have.  And
using the same test machine to test multiple projects concurrently becomes
unthinkable.

Dockers solve all of the above problems in one shot.

## What is Docker?

Docker is an OS-level virtualization software that allows you to run a
light-weight Linux virtual machine with all necessary software pre-installed on
the host OS.  The host OS can be either Linux, MacOS, or Windows.  The
virtualized execution environment that software runs in is called a "Docker
container".  The read-only template used to build containers is called a "Docker
image".  Images are read-only so any changes to the file system or otherwise
done to the execution environment in the container does not modify the image ---
they are discarded when the container terminates.  This is a very useful
property for us since it means that you can recreate the exact same execution
environment every time you create a container, ensuring reproducibility.

## How are we going to use Docker?

We are going to use Docker in two ways for our testing purposes:

1. We are going to create a Docker image containing our microservice along with
   the web server we want to run it on.  We are going to use the Flask web
framework along with the included web server.

1. We are going to download a pre-created Docker image of the PostgreSQL
   database server from [Docker Hub][6].

Each time we run our test, we are going to create two containers from these two
images.  The two servers in the two containers will communicate over the network
through a TCP/IP port just like as they would in a real system.  There is a tool
in Docker that is designed specifically for this scenario where you have to
launch a system that is composed of multiple containers called "Docker compose"
that you will use later on.

The advantage of course is that it does not matter whether Flask or PostgreSQL
is installed on the host machine, or what state they are in.  The tests will run
uniformly no matter what.  You can even have multiple tests of multiple projects
using different versions of Flask and PostgreSQL run concurrently without
stepping on each others' toes.

## What does the microservice do?

A microservice is an application that provides a "small" service.  This service
can be as simple as providing a bunch of string operations such as the ones in
simple_strop that we tested in Part 1.  In fact, simple_strop is exactly what we
are going to build our example microservice out of.  The service is presented to
the outside world as a bunch of APIs accessible through the internet, typically
using the HTTP protocol.  JSON (JavaScript Object Notation) is often used to
format data that is passed to and from the API endpoints.  So a microservice is
essentially a web application hosted on a web server, but is used to service
data requests rather than present a viewable web page to the end user.

This particular microservice is written in python using the [Flask][5] web
framework.  In addition to the two main functions (`reverse` and `piglatin`)
from simple_strop, the web server provides a simple authentication system (not
safe for use in a production environment without improvements outside of the
scope of this application) that will allow us to save the history of commands a
user sent to our server.

Here is an example interaction with the microservice:

```python
>>> from requests import get, post
>>> get('http://localhost:80/').json()
{'message': 'System online.'}
>>> post('http://localhost:80/register', json={'username':'wonsun', 'password':'password'}).json()
{'message': 'User "wonsun" registered succesfully.'}
>>> post('http://localhost:80/authenticate', json={'username':'wonsun', 'password':'password'}).json()
{'token': '916c22895064c363b03634d580da0762'}
>>> post('http://localhost:80/reverse', headers={'Authorization':'916c22895064c363b03634d580da0762'}, json={'input': 'This is my string to reverse'}).json()
{'output': 'esrever ot gnirts ym si sihT'}
>>> post('http://localhost:80/piglatin', headers={'Authorization':'916c22895064c363b03634d580da0762'}, json={'input': 'This is my string to convert to piglatin'}).json()
{'output': 'Isthay isway myay ingstray otay onvertcay otay iglatinpay'}
>>> get('http://localhost:80/get-history', headers={'Authorization':'916c22895064c363b03634d580da0762'}).json()
{'history': [{'function': 'reverse', 'input': 'This is my string to reverse', 'output': 'esrever ot gnirts ym si sihT', 'time': 'Sat, 10 Apr 2021 19:59:53 GMT', 'user': 'wonsun'}, {'function': 'piglatin', 'input': 'This is my string to convert to piglatin', 'output': 'Isthay isway myay ingstray otay onvertcay otay iglatinpay', 'time': 'Sat, 10 Apr 2021 20:00:51 GMT', 'user': 'wonsun'}]}
```

As you can see, this server accepts either a GET or POST request (depending on
the endpoint) with all of the arguments specified in the request JSON.  If you
want to learn more about making HTTP requests via python, I recommend reading
the python package [requests][1].  It is with that package that I prepared the
example here:

# Steps to integrate a change and deploy it

Just like in Part 1 of the exercise, we want to create a series of steps that
culminate in deployment.  In this case, it is going to happen in three steps: 1)
Setting up the build environment, 2) Testing, and 3) Distributing.  Also, in
Part 1, we distributed our software by uploading a Python package to PyPI.  For
Part 2, we are going to create a Docker image of our microservice and push it to
GitLab's repository-specific registry as part of the distribution step.  The
registry is accessible through `Packages & Registries > Container Registry` on
your GitLab repository page.  If you want a much more public distribution, look
into [publishing an image on Docker Hub][8] as this is the de facto standard for
making your images publicly accessible and searchable.

## Setting up the Build Environment

This repository once again makes use of pipenv for explicit environment control.
If you are unfamiliar, [see the documentation in this repository][4]. When you
are getting started, make sure you enter a new virtual environment using:
```
pipenv shell
```
And make sure that you install all necessary packages from `Pipfile.lock` with:
```
pipenv install --dev
```

## Testing

For this demo, the tests we have are different in nature to the ones from the
`simple_strop` package. In that example, we were concerned only with unit testing the
logic of individual functions that made very few (if any) calls to external
code. Here, we want to perform integration and systems testing for the entire
system including the Flask web server, and the PostgreSQL database server.

We are going to use the same `pytest` framework that we used for Part 1.  In
this repository, you can find the tests written in the three files in the
`tests` directory. Note that the `__init__.py` file includes the definition of
pytest fixtures that we reference in both our tests for views and for the utils
module.

Within the pipenv shell, try running:
```
pytest
```

You should see all tests (except 1) failing:
```
...
================================================================ 1 passed, 12 errors in 78.56s (0:01:18) =================================================================
```

Why? That is because neither the web server nor the database server is currently
running, and without them, nothing works!

#### Running the Webserver and Docker Compose

Like most Flask web servers, the development server can be run using the
command:

```
flask run
```

This will start the server, claim a port (in this instance port 5000), and
listen for requests. You can start the server like this, then in your browser go
to [http://localhost:5000/](http://localhost:5000/){target="\_blank"} to see
that it is working. You should see something like this:

```
{
  "message": "System online."
}
```

Unfortunately, this isn't all that must be done. Because the web server assumes
its access to a database, as soon as you perform some operation that requires
access to the database, the server will crash rather spectacularly. What we need
now is to give the server a database to use and we'll accomplish this via
docker-compose.

To use docker-compose is simple, with docker installed on your machine and the
docker daemon running, simply run `docker-compose <command>`. You might choose
`build`, `up`, `down`, `pull`, or any other of the many commands that
docker-compose provides. When you run this command, the docker-compose
application will look in the directory where the command was executed for a file
called `docker-compose.yaml`. This is the default name for docker-compose files,
but if you want to reference a different file (e.g. dev.docker-compose.yaml or
test.docker-compose.yaml), you can specify that with the `-f <filename>`
argument.

In our repo, we only have one docker-compose file, so just running:
```
docker-compose build
```

will suffice. This will take about 3 minutes to complete depending on whether or
not you've run the command before for this application (it caches very well),
whether you have the base images, and your machine. This command prepares all of
the images (remember that images are the virtual machines we talked about in the
intro) referenced in the compose file to be run as containers (the running
version of an image).

#### The Dockerfile

Images are defined using a file format called the "Dockerfile". It has a very
simple syntax and you'll probably only ever need to know a few directives:
`FROM`, `RUN`, `COPY`, `WORKDIR`, and `CMD`. More complex Dockerfiles can make
use of directives like `ENTRYPOINT` and `ENV`, but this will do for our example.
As I describe each of these directives, I encourage you to take a look at the
Dockerfile in this repository to see an example of what I'm talking about.

The `FROM` directive describes where the Dockerfile should start and the
argument is the name of an existing image and tag. Tags are a way to
differentiate versions of an image. For example, the python image has tags for
3.7, 3.8, 3.9, more specific versions like 3.9.4, versions with different
operating systems as bases like 3.9.4-slim, and many more. You can find a wealth
of public and free to use images on [Docker Hub][6].

The `RUN` directive is for executing a command inside of the image you're
building. This might be to install runtime or buildtime dependencies into the
image, to create a certain filestructure, or anything else you can dream of.
Images are not just virtual machines for simulating an operating system, they're
pared down virtual machines for executing an application in a very specific
environment. The `RUN` directive helps you define what that environment looks
like. In our example, we use the `RUN` directive to install `libpg-dev`, `gcc`,
and other buildtime dependencies for our application. Without these
dependencies, we won't be able to install the python packages we need later that
are written partially or entirely in C or reference special C libraries just for
that application.

The `WORKDIR` directive tells docker which directory you are currently working
in. Think of this as the docker equivalent of `cd` when you're changing
directories on your machine, but Docker also remembers where you were. If I
define `WORKDIR /home/user/documents`, then when I run any commands in the
container from this image, those commands will run in that directory (unless
otherwise told not to). In our example, we use a directory called `/app` (which
is somewhat of a Docker standard) to define where our application lives and
runs. For the most part, we shouldn't need to affect the file system anywhere
outside of this directory.

The `COPY` directive is for moving files from outside the container to the
inside or between containers. This is how we copy in our requirements.txt file
to install our packages in pip. Note: this is how we can make use of pipenv to
get pip to install very specific packages. By using `pipenv lock -r`, we can
generate a `requirements.txt` file that describes all of the packages we need
and the very specific versions we're expecting. We can then have pip install
this list of packages directly from the file using the `-r requirements.txt`
argument. We can also copy files between images. In our Dockerfile, we're using
a technique called "multi-stage builds". This helps us to keep our images as
small as possible by defining a build stage where all of the runtime
dependencies are created, then defining an application stage where we copy over
those runtime dependencies without needing to build them locally. This prevents
things like build caches, log files, and other extraneous build artifacts from
lingering in our images and increasing the size. Multi-stage builds might also
improve the speed at which your images build because each stage is built
simultaneously until it requires something from another stage. In order to copy
files from one stage to another, we must use the `as` keyword in from to give
our stages names, then use the `--from` argument with `COPY` to specify which
stage to look in.

The `CMD` directive is for defining what a container for this image should
default to running. In our example, we want our container to start the server by
default, so our `CMD` in the Dockerfile is the command to do that. Note that
even though outside of the container we can run our server with `flask run`, we
can't make the same assumptions in the container. The commands we run in the
container don't assume we have a shell, so if a command isn't in PATH, we aren't
going to be able to use it directly. Here, we first specify to run a shell (bash
to be specific), then pass it the arguments (-c) `flask run --host 0.0.0.0`. The
`--host` argument to the `flask run` command is for telling flask where to
listen. When we ran this command outside of the container, we wanted to access
our server on localhost (or it's formal name `127.0.0.1`) and that's what flask
defaults to so we didn't need to specify the host. When we run the server in the
container, however, we are now running the server on a "different machine" as
far as flask is concerned, so we don't have access to the same localhost. To
flask running in the container, it will look like we're accessing the server
from some strange network address. The `0.0.0.0` is a networking alias (called a
mask) that says "I don't care where the request is coming from, if it got routed
to you allow it".

#### Using Docker Compose with our Dockerfile

Now that we understand what our Dockerfile is generally doing, we can start to
examine how we can use docker-compose to make our lives a little easier. If you
look in our `docker-compose.yaml` file, you'll see that under services we define
two keys: `psql` and `server`. These are the two parts of the application:
`psql` is the common abbreviation for PostgreSQL and `server` is exactly what
we've made.

For each service, you can see that we defined either an image or build value.
When we say "image", we're telling docker-compose to create this service from an
existing image on Docker Hub (unless specifying another registry). Here, we're
using the public image for PostgreSQL version 13.2. We also load the environment
variables for our image from the `.env` file so that we can configure the
database created by the postgres image to use a certain name, have a certain
password, and so on. Almost all public images have some sort of documentation
with them that describes how to use the image and [postgres is no exception][7]
(check out the "How to use this image" section). The last thing we define for
our `psql` service are a set of ports in the format `<external>:<internal>`
relative to the image. So what we're saying is "when the host machine receives
traffic on port 5432, send that into the container on port 5432". More on this
when we get to the server.

Now let's take a look at our `server` service. Instead of an `image`, here we
define how to build this service. We specify the build context ("." means "same
directory as the docker-compose.yaml") and which dockerfile to use (in case we
had a bunch like "dev.Dockerfile", "test.Dockerfile", etc.). We also give this
container a name: "flask". This allows us to more easily reference this
container if we wanted to use some docker command on it while it was running
like restarting it, opening a shell into it, etc. We also override the command
from our Dockerfile here. We don't have to do this because the Dockerfile
already has a command specified, so we could just remove the `command` section
entirely. In our case, we don't want to run `flask init-db` by default in our
Dockerfile because if someone ran the container without setting up the database,
that command would crash on startup. Here with the docker compose, we know we
have the database service already so we can run this command without concern. We
also load in the same `.env` file for our environment variables (having both
services pull these values from the same place means we can update them in one
spot), but we override some of the values that differ between when we try to run
from the command line and when we try to run in docker-compose. In
docker-compose, services are accessible at a url that looks like
`http://<servicename>:<serviceport>/`, so that is obviously different from the
localhost that we need if we were running this outside of a container. Here
again, we define ports but note the difference: we're mapping the external port
80 to the internal port 5000. Recall that when we ran our webserver from the
command line, we had to access it with "http://localhost:5000". What this is
saying is "when you receive traffic on port 80, send it into the container at
port 5000". Because port 80 is the default port for unencrypted web traffic (443
is for encrypted), we can access our webserver running in docker-compose with
`http://localhost/` (or `http://localhost:80/` if we want to be specific).
Finally, we use the `depends_on` key to specify what services need to be running
before this service can start. Here, the database needs to be up before we use
our webserver, so we specify the database service.

Now that we understand what our docker-compose is saying, how can we use it? By
running `docker-compose build`, we tell the docker-compose application to build
any of the images with a `build` section defined. We can then run:

```
docker-compose up
```

to start the containers from each of the services, setup a virtual network so
they can talk to each other, and open the port mappings so we can access them!
Now, if you run your `Docker Desktop` GUI application and click on `Images`, you
should see two images: `dockerized-server_server` (the Flask server image that
you built) and `postgres` (the PostgreSQL server image that you pulled from
Docker Hub).  Also, if you click on `Container / Apps`, you should see
`dockerized-server` running with the `flask` and `dockerized_server_psql_1`
containers running under it. 

Note, you can also use `docker-compose up --build` to build the images, then
start them, `docker-compose pull` to pull any images that aren't built locally,
and `docker-compose down` to remove all of the containers, network, etc. that is
stored on your machine to free up space.

Now, we can finally access our server! You can open the python repl by just
running `python` and try entering in the commands from the example usage at the
top of this document.  Also, if you run:

```
pytest
```

then all tests should pass:

```
========================================================================== test session starts ===========================================================================
platform win32 -- Python 3.9.4, pytest-6.2.3, py-1.10.0, pluggy-0.13.1
rootdir: C:\Users\mrabb\Desktop\Wonsun Ahn\cs1632\gitlab_exercises\dockerized-server, configfile: pytest.ini, testpaths: tests
plugins: Faker-8.0.0, cov-2.11.1
collected 13 items

tests\test_utils.py .......                                                                                                                                         [ 53%]
tests\test_views.py ......                                                                                                                                          [100%]

----------- coverage: platform win32, python 3.9.4-final-0 -----------
Coverage HTML written to dir htmlcov


========================================================================== 13 passed in 24.30s ===========================================================================
```

## Distributing

The easiest way to publish the Flask image that you just built to the GitLab
repository-specific registry is through using a GitLab pipeline, so I won't go
into manually uploading an image.  Again, you can look into [publishing an image on Docker
Hub][8], if you want to learn more about publishing images.

# Automating CI/CD using Pipelines

Now we are going to automate all the CI/CD steps we described using pipelines.

In the following sections, I'm going to assume you have a basic understanding of
pipelines.  If you feel you need to refresh your memory, please take a look at the
[pipelines section in Part 1][9].

## The `build` Stage

Comparing with the previous example, notice that the `build` stage is identical.
We are still going to be building and testing our application in the pipeline
runner's local environment, so we'll need all of our dependencies. To be clear,
note that the first two lines of the pipeline are telling the pipeline to run
all of our commands inside of a python version 3.9 docker image. So when we say
we're running in the "runner's" local environment, we're actually running in
_exactly_ the same environment as if we pulled the official python 3.9 image
from Docker Hub and ran all of these commands inside of it instead.

## The `test` Stage

Just like last time, after the `build` stage is the `test` stage. Because all of
our test and coverage configuration is saved in our `.coveragerc` and
`pytest.ini` files, we can again run everything with just `python -m pytest`.
This time though, we have a few extra fields:

```yaml
variables:
    FLASK_APP: flaskr
    FLASK_ENV: development
    POSTGRES_DB: db
    POSTGRES_USER: user
    POSTGRES_PASSWORD: password
    POSTGRES_PORT: 5432
    POSTGRES_HOST: postgres
services:
    - postgres:13.2
```

Recall that when we were taking a look at our docker compose configuration, we
described our environment variables as coming from the `.env` file. When we were
working in our local virtual environments, these variables were put into our
entironments automatically by pipenv (which automatically adds variables to your
environment from a `.env` file when you enter a virtual environment). In our
pipelines, we don't have the luxury of letting pipenv place our environment
variables, so we do it ourselves through the `variables` key. Each of these
values will be placed into the environment of any container and/or service
running for this job. That includes the container where we run the `pytest`
command, but it also includes the postgres service we connect to for the tests.

In gitlab pipelines, services are additional running containers. You can't
specify code to run inside of these containers, but you can specify environment
variables. Services are ideal for things like test databases that don't need to
persist and just need to be around long enough for one job. As we saw eariler,
in order to use the postgres container, we specify the `POSTGRES_...`
environment variables so that we know what the connection details are. Just this
small section is enough to tell the pipeline to start the postgres container,
put the environment variables inside, and leave it running until our unit tests
are done. The rest of the testing stage is the same, so I won't cover it here,
but again we are able to simply run the command `python -m pytest` and we're
good to go.

## The `dockerize` Stage

This is the deployment stage.  Finally, after testing our code, we're ready to
dockerize/containerize/build the image (synonyms) and push it to our registry.
This stage looks complicated, and in some ways it is, but let's take a look at
it from the perspective of how you would do the same thing on local. Notice, we
have 3 commands in our script.  These are analogous to logging into the docker
service, building the image, and pushing the image.

The first command may be familiar to you if you have used linux before: it is
simply making a directory. The `-p` argument stands for "parent" and means "if
any directories along this path don't exist, create them too".

The second command is the equivalent of logging into the docker service. On your
local machine, you would do something like this with `docker login`. Because
this login process is interactive, we can't do that exactly in the pipeline, but
we can build the result directly. Docker stores login credentials in a json
format in a file called `config.json`. The format of this file is as follows:

```
{
	"auths": {
		"<registry>": {
			"username": "<username>",
			"password": "<password>"
		}
	}
}
```

Because we know this format, we can just create it directly using the common
unix idiom `echo <file_content> > <filename>`. We get the values of each of
these from [GitLab's set of pre-defined variables for CI/CD pipelines][11].
GitLab has dozens of variables that you can use and it helps abstract away a lot
of the strangeness of working inside of containers.

Then, the last command uses the [kaniko executor (provided by google)][10] and
does the duty of both building and pushing the image to the specified location.
Up until this moment, I haven't mentioned kaniko and that is for one very
important reason: it is complicated and atypical. Kaniko is the solution to a
problem called "docker-in-docker": the idea that within a docker container, we
wish to build, push, pull, or start another image/container. You can see how
doing this could wreak havoc on a system's resources if it were to recurse
indefinitely, so in order to prevent that situation it is somewhat difficult to
do. Kaniko gets around that by providing an application layer between the user's
intent and the system which allows for error checking, limiting the scope of the
user's actions, etc. We have to use kaniko because the pipeline runners are
accomplishing all of these tasks _within containers_. If these commands were
just running on some random host machine, we wouldn't have to worry about
docker-in-docker, but we would have to spend much more time on security and
access control and concurrency and all of these other problems that docker waves
away through virtualization.

The kaniko command we're using is the executor and this allows us to both build
and push our image to where it belongs. The command is relatively simple:
`executor --context <the_directory_where_the_Dockerfile_is> --dockerfile
<the_dockerfile_to_use> --destination <where_to_push_the_image>`. We can use
some pre-defined environment variables from GitLab to make this much simpler:
where is our build context? The top level of our repository, so we can specify
that location with $CI_PROJECT_DIR. Where is our dockerfile? Well it's in the
top level of our directory, so we can specify that location with
`$CI_PROJECT_DIR/Dockerfile`. Finally, where do we want our image to be pushed
to and what version tag do we want on it? We want to push it to the default
repository for our project (`$CI_REGISTRY_IMAGE`) with the name `$IMAGE_NAME`
(defined in our variables section for this job) and the for ease of version
tracking, let's have the build tag just be the first 8 characters of the commit
that this pipeline is run for (`$CI_COMMIT_SHORT_SHA`). Those together give us
our command as you can find it in `.gitlab-ci.yml`.

With that, our newly built image has been pushed and we can find it in that same
container registry page on GitLab. There you'll also find instructions to pull
that image, so you could pull it down and run it locally or give that image path
to someone else and they can pull it on their machine and through the magic of
Docker, the microservice contained in that image will behave in _exactly_ the
same way.

## Testing the deployed image

Let us try pulling the newly deployed image on to our local machine and test it
ourselves.  For this purpose, I have created a
`docker-compose-using-registry.yaml` file that is identical to the original
`docker-compose.yaml` file except that now it downloads the new image from the
GitLab container registry instead of building it.  Note how the `build` section
on the `server` service has been replaced by an `image` with a URL to the image
in the container registry.  You will have to edit the URL to reflect the true
URL of your image in the registry as shown in:

https://docs.gitlab.com/ee/user/packages/container_registry/

Also, the registry is made available to the outside world only if your
repository is public.  So make sure it is by checking `Settings > General >
Visibility, project features, permissions` and confirming that `Project
visibility` is set to `Public`.  If it is not, change it to be so and `Save
Changes`.

Once you have this set up, first kill all containers that are currently
running and make sure they are all cleaned up by doing:

```
docker-compose down
```

Confirm that your local Docker console shows no containers running.  Now it
is time to try launching the new image using the following command:

```
docker-compose -f docker-compose-using-registry.yaml up
```

That will launch the new registry image along with the PostgreSQL image as
before, and you will be able to access the webserver through port 80.  Now
let's try using pytest in our pipenv shell like before to check that all
tests pass:

```
pytest
```

And they should, verifying that this new registry image is a fully
functioning Flask application that is ready for deployment.

[1]: https://pypi.org/project/requests/
[3]: https://docs.docker.com/get-docker/
[4]: https://gitlab.com/wonsun.ahn/simple-python-package#short-intro-to-pipenv
[5]: https://flask.palletsprojects.com/en/1.1.x/
[6]: https://hub.docker.com/
[7]: https://hub.docker.com/_/postgres
[8]: https://docs.docker.com/docker-hub/publish/publish/
[9]: https://gitlab.com/wonsun.ahn/simple-python-package#pipelines
[10]: https://github.com/GoogleContainerTools/kaniko
[11]: https://docs.gitlab.com/ee/ci/variables/predefined_variables.html
