- [Introduction](#introduction)
  * [What does this package do?](#what-does-this-package-do)
- [Steps to integrate a change and deploy it](#steps-to-integrate-a-change-and-deploy-it)
  * [Setting up the Build Environment](#setting-up-the-build-environment)
      - [Short Intro to Pipenv](#short-intro-to-pipenv)
  * [Testing](#testing)
  * [Packaging](#packaging)
  * [Distributing](#distributing)
- [Automating CI/CD using Pipelines](#automating-ci-cd-using-pipelines)
  * [Pipelines](#pipelines)
  * [The `build` Stage](#the-build-stage)
  * [The `test` Stage](#the-test-stage)
  * [The `package` Stage](#the-package-stage)
  * [The `deploy` Stage](#the-deploy-stage)
  * [Invoking the Pipeline](#invoking-the-pipeline)

# Introduction

The goal of this exercise is to create a CI/CD (Continuous Integration /
Continuous Delivery) pipeline on a GitLab repository such that a source code
push on the repository automatically triggers a pipeline that performs:

1. Integration: Making sure the source code compiles and all unit tests pass before allowing the push to be merged.

2. Delivery: Packaging the code into a distributable (called a "wheel" in
   Python) and deploying it to the end-users (the Python Package Index
pypi.org repository in case of Python).

The unit tests (an potentially other tests) in the automated pipeline
provide the quality control and confidence necessary to allow developers to
push code to the central repository directly and have it immediately
delivered to the end users.  That is why it is often said that Continuous
Testing (CT) is a prerequisite for CI and CD.

Having a single repository for the entire organization cuts down on
operating costs that comes from having to maintain multiple "beta branches"
for the multiple beta feature developments going on in the organization, and
having to merge those branches into one deployment version on delivery
dates.  The difficulty of merging these branches is often the reason for
delivery delays.

## What does this package do?

You can install this package from the python package manager `pip` with `pip install simple-strop`. Once installed, you should be able to see package listed when you do `pip list`.  The package has 2 modules (that aren't tests): `operations.py` and `utils.py`. An example usage is below:

```python
$ python
>>> import simple_strop as ss
>>> sample_string = 'This is my sample string'
>>> ss.reverse(sample_string)
'gnirts elpmas ym si sihT'
>>> ss.piglatin(sample_string)
'Isthay isway myay amplesay ingstray'
>>> ss.is_capitalized(sample_string)
True
>>> ss.is_all_caps(sample_string)
False
>>> ss.is_vowel('y')
False
>>> ss.is_consonant('q')
True
>>> exit()
```

So where did the simple-strop package come from?  It came from the Python
Package Index (PyPI) repository aforementioned.  The URL for the
simple-strop package in PyPI is: https://pypi.org/project/simple-strop/.

By the end of this exercise, you will have learned how to creat a CI/CD
pipeline that builds a package of your own and deploys it to PyPI.

# Steps to integrate a change and deploy it

On every source code modification, the following steps must happen to have
CI/CD: 1) Setting up the build environment, 2) Testing, 3) Packaging, and 4)
Distributing.  We will describe each step in more detail using our example.
Later, we will talk about how to automate all these steps as part of a pipeline
using GitLab.

## Setting up the Build Environment

One of the most frequent issues when it comes to porting software from one
machine to another is ensuring that the two machines _agree completely_ on what
the environment for building that code is. When I say "environment", I'm talking
about the variables that are set on the system as well as the specific versions
of packages that are installed on the system.

If you're running a Linux distribution and I'm running on Windows and our
mutual friend is running on MacOS, then there is no chance that we can have
_all_ of our environment variables match (without some sort of
virtualization... foreshadowing), but we can ensure that our package versions
are identical by using a virtual environment manager like [pipenv][1]. Pipenv
is an easy tool for ensuring that not only do you have the same packages
installed, but that you have the same minute versions of those packages
installed. It takes a little bit of getting used to, but once you've adopted it
the benefits are immense!

Once you start using pipenv, you also have the benefit that your list of
globally installed packages (the packages that you install when you just run
`pip install ...`) will become much shorter.

Pipenv is just another python package and can be installed with `pip install pipenv`.

#### Short Intro to Pipenv

There are a few bread-and-butter commands, so I'll cover those first:

| Command                           | Description                                                                                                                                                                                                                                                                                                                          |
| --------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `pipenv shell`                    | This enters your shell into the virtual environment or creates an empty one if the directory you're in doesn't contain a Pipfile.                                                                                                                                                                                                    |
| `pipenv install <package_name>`   | This installs a package into your environment and creates a new environment if the directory you're in doesn't container a Pipfile. If a Pipfile exists but there is no virtual environment stored on your machine, pipenv creates a virtual environment and populates it with packages from the Pipfile.lock.                       |
| `pipenv lock`                     | This resolves all of the dependencies of all of the packages you've installed and chooses the most modern versions that don't conflict with other package dependencies. It can't be used unless you have a Pipfile. You can also use it to generate your `requirements.txt`                                                          |
| `pipenv uninstall <package_name>` | Remove a pacakge from your environment. Can't be used unless you have a Pipfile (with whatever package you're trying to remove)                                                                                                                                                                                                      |
| `pipenv --rm`                     | Deletes the virtual environment for this directory, but keeps the Pipfile and Pipfile.lock. Useful if you've installed extraneous packages into this environment without using pipenv (like `pip install ...` inside of the pipenv shell) or if you are done working on a project for the time being and want to free up some space. |

So to get started, you go to the top level of whatever git repository you're using for your code and run `pipenv shell`. This will create and enter a virtual environment that is completely distinct from your base environment. You can install any packages you like without cluttering up your home environment with `pip` or you can add pacakges to your project with `pipenv install`.

For this project, I added 5 packages: 2 build-time dependencies - build, twine; and 3 development-dependencies - pytest, pytest-cov, coverage. To install run/build time dependencies just use `pipenv install ...`, but if you want to install dependencies that are manually enabled, use `pipenv install --dev ...`. The added benefit is when generating requirements.txt files, your dev packages won't be included (by default).

After you install packages with pipenv, it will automatically generate a `Pipfile` and a `Pipfile.lock` (although you can tell pipenv not to lock). The Pipfile is basically just a meta description of what your project needs. It says things like what version of python the project is running in, what the packages are, what the dev dependencies are, and then some information about where pipenv was installed from. The Pipfile.lock is more specific and contains a hash of all of the packages for your project, hashes for verifying the installation of those packages, the specific version numbers of the packages, the list of dependencies for each installed package, hashes for those, their dependencies, etc.

**YOU SHOULD NEVER ATTEMPT TO MANUALLY EDIT YOUR PIPFILE.LOCK**. In the worst case, it might make your environment uninstallable and you will need to remove the lock file (which will forget all of your pinned versions) in order to correct this. Then you might have to deal with situations where new versions of your dependencies were released that break your code... it's bad. Let pipenv handle the lock file and get familiar with commands arguments for the install, uninstall, and lock commands.

## Testing

For this exercise, we'll focus on unit tests for this package. As you can see
in the directory, the tests are right along side the normal code in this
package. This was a choice I made out of convenience because I wanted to do the
minimal amount of setup with pytest and coverage and if I were to try and
separate the tests from the source, it would create import complications and
the coverage would be wrong (by default) and it would just be a whole mess.

You can run the unit tests yourself locally. First install the environment via
pipenv: `pipenv install --dev`, then you can just run the `pytest` command. If
you don't include the `--dev` flag, you won't install the packages for unit
testing and it won't work.

Pytest is configured to also run a code coverage report and save that into a
directory called `htmlcov`. After running the unit tests, open the file
`htmlcov/index.html` with a web browser and you'll be able to interactively
click through each file, see which lines are covered (spoiler alert, it's all
of them), and see what the coverage of the whole project is.

You can get familiar with how coverage is working by commenting out some of the
functions in the `src/simple_strop/test_...` files then running `pytest` again
to see the coverage re-calculated.

## Packaging

For the details on how to create a Python package, I'll direct you [to this
guide][3].  I created the Python package myself using this guide.  You will see
in the guide that the `setup.cfg` contains configurable metadata about the
package.  I'm going to ask you to change the package name from:
```
name = simple-strop-wonsun.ahn
```
to (where your.name is your actual name):
```
name = simple-strop-<your.name>
```

This is important since each of you need a unique package name to register on
PyPI.  Otherwise, PyPI will complain of a name conflict.  For the same reason,
if you attempt to register the same package twice to PyPI, PyPI will complain.
So what should you do if you want to update an existing package?  You need to
increment the version number.

Once you are done editing `setup.cfg`, go ahead and run the build package command in the pipenv virtual environment:
```
python -m build
```

This will run for a few seconds and when it's done, you'll have a few new
directories: `build/`, `dist/`, `src/simple_strop.<your.name>.egg-info/`. Each of these
serves its own purpose, but the `dist` directory is how we might actually
_distribute_ our package.

Inside of `dist` there should be two files:

- `simple-strop-<your.name>-<version>.tar.gz`
- `simple_strop-<your.name>-<version>-py3-none-any.whl`

The first file is just a compressed archive of the files in this repository
that are actually meaningful to the distribution of your project. If you'd
like, you can open this file (on linux and mac machines natively, windows if
you have gnu installed) with `tar xf <filename>` and then check out the new
directory it created with the same name as the tarball.

The second file is a "wheel". Wheel is python's format for creating installable
packages. It's a binary file that contains the source code and instructions to
pip (or pipenv) on how to actually install the package, what additional needs
it has (like adding a file to the user's path like pipenv does), etc.

You can actually install packages directly from wheel files with `pip install <path_to_wheel>`

## Distributing

Once you've built the package into `dist`, you can upload it to a package
directory like PyPI!  But we probably shouldn't clutter up the PyPI with dozens
of copies of the same package from dozens of students.  There is actually a
test version of PyPI that developers can use to play around or make sure that
their package is actually all set for distribution _before_ they publicly
publish it. [You can find TestPyPI here][2].

The command to upload to TestPyPI is:
```
python -m twine upload -r testpypi dist/*
```

If you wanted to upload to PyPI, you would omit the `-r testpypi`
(discouraged).  You'll need to create an account with TestPyPI or PyPI to be
able to upload packages to either.

# Automating CI/CD using Pipelines

Now we are going to automate all the CI/CD steps we described using pipelines.

## Pipelines

The last component of this repository is the CI/CD pipeline. Pipelines are a
sequence of steps organized into a DAG (directed acyclic graph) for managing
builds, tests, releases, and anything else that a project might need to go from
source code to distributable. You can see the configuration for this
repository's pipeline in the file `.gitlab-ci.yml` in the top level of the
repository. The `.yml` extension identifies this file as a "yaml" file, almost
all pipelines are defined in yaml files as they are easy to read and are quite
flexible. [You can read more about yamls here][4].

At a minimum, a pipeline should have 2 stages: build and deploy (but test is
also highly recommended!!!). In this repo, we actually have 4 because we need
to build two times: first to build the dependencies that our pipeline will need
for testing and second to actually build our source code into the wheel file.

The stages of the pipeline are defined at the top:

```yaml
stages:
  - build
  - test
  - package
  - deploy
```

Each of these will fulfill one part of the process and the final one should
actually make our package public. So let's dig into each one and understand
what's happening.

## The `build` Stage

Our build stage only has 1 job in it. A job is a single, focused collection of
commands to accomplish a specific goal. It is more focused than stages. This
job is called `build-dependencies` and it is responsible for installing the
packages we need for the later stages.

In this stage, we see the first occurrence of a "yaml anchor". Anchors can get
pretty complicated, but the easiest way to think about them are as variables
but for yaml files: you define a variable with `&variable_name` and you can
then reference it with `*variable_name`. So when we see:

```yaml
.cache: &cache
  key: $CI_COMMIT_REF_SLUG
  policy: pull
  paths:
    - $PIP_CACHE_DIR
    - $PYTHON_PACKAGE_DIR

build-dependencies:
  stage: build
  cache:
    <<: *cache
    policy: pull-push
```

we are defining the anchor `cache` at the top to be the object containing three
keys: `key`, `policy`, and `paths` where `paths` is an array with two indices.
We then use our `cache` anchor inside of the `build-dependencies` object with
the "merge operator" `<<: *variable_name`.

This merge operator is saying, "I want to put the value of this anchor in this
position as if it were defined here". So after the merge operator, if we
expanded this file it would look like this:

```yaml
.cache:
  key: $CI_COMMIT_REF_SLUG
  policy: pull
  paths:
    - $PIP_CACHE_DIR
    - $PYTHON_PACKAGE_DIR

build-dependencies:
  stage: build
  cache:
    key: $CI_COMMIT_REF_SLUG
    policy: pull
    paths:
      - $PIP_CACHE_DIR
      - $PYTHON_PACKAGE_DIR
    policy: pull-push
```

Notice that the key "policy" is repeated. In this instance, we want to override
it from what is defined in our anchor, so we can just define it again and yaml
will forget the previous value.

Inside of the `build-dependencies` job, we can see that we have defined it to
be in the stage "build". We also define how we want the cache to work. Defining
a cache isn't necessary, but it is useful for speeding up repeated builds. In
this stage, however, we don't want to use the cache, we want to overwrite the
cache so that it is fresh for this build. Here, we identify a reusable cache by
the git commit the pipeline was run for (`CI_COMMIT_REF_SLUG`) and say that we
want to be able to push to this cache.

We also define our "artifacts". Artifacts are files, strings, or directories
that are the output of a job and are needed for another job or just for later
reference. In this case, after we install our packages, we want to keep the
directories where we installed these packages for use in the future stages, so
we define those directories as artifacts.

Finally, we define our scripts. Before executing the standard script, we make
sure to delete our old cache and add our cache to our PYTHONPATH environment
variable. This tells python how to find these packages later. Then we run two
`pip install` commands to install the packages that we defined in our
requirements files. Note the usage of variables with `${VARIABLE_NAME}`. This
allows us to change these values easily in one spot (the variables section) and
not have to worry about tracking down all of the places in the pipeline we used
them. GitLab's pipelines support variables in almost any value for key value
pairs and as environment variables in scripts.

## The `test` Stage

```yaml
test:
  stage: test
  cache:
    <<: *cache
  artifacts:
    expire_in: 1 day
    paths:
      - htmlcov
      - .coverage
  coverage: '/Total coverage: \d+.\d+%$/'
  script:
    - python -m pytest
```

Now that we've built our dependencies, we're ready to run our unit tests. You
can see that most of the configuration here is very similar to the build stage:
we pull in our cache anchor (without overriding the default policy), we define
the coverage output as artifacts, and we run a script. The script here is just
`python -m pytest` because we already defined `.coveragerc` and `pytest.ini` to
run our tests with all of the command line flags and configurations that we
need. Note: when we were running locally, we ran with just `pytest`. The
command here is logically identical, but a little more specific and helps us
avoid issues with caching in pipelines.

Note the inclusion of the `coverage` line. That is how we tell gitlab to parse
the output of this job with the regular expression so that the pipeline can
report our code coverage at the end.

## The `package` Stage

```yaml
build-package:
  stage: package
  cache:
    <<: *cache
  artifacts:
    expire_in: 1 day
    paths:
      - dist
  script:
    - python -m build
```

This is pretty similar to the previous stages as well, but here we are defining
our `dist` directory as an artifact. We want to make sure that in the final
stage, we access it to deploy. We have this stage defined on its own for a few
reasons: first, we want it to benefit from the cache because it would be very
annoying to have to rebuild this late in the pipeline; second, we don't want to
spend the time actually building until we've tested and know that everything is
working correctly.

## The `deploy` Stage

```yaml
.release:
  stage: deploy
  when: manual
  cache:
    <<: *cache
  dependencies:
    - build-package

release to testpypi:
  extends: .release
  script:
    - python -m twine upload -u ${TEST_PYPI_USERNAME} -p ${TEST_PYPI_PASSWORD} -r testpypi dist/*

release to pypi:
  extends: .release
  script:
    - python -m twine upload -u ${PYPI_USERNAME} -p ${PYPI_PASSWORD} dist/*
```

Finally, we've built and tested our package and we're ready to hand it off the
the world! It's time to introduce one final concept of pipelines: templating.
Templates and anchors share the same basic goal to reduce repetition, but they
accomplish it in different ways. Anchors are single-file _only_. If I defined a
pipeline where one file pulled configuration from another, then I couldn't
reuse the anchors across files. Also, it looks a little cleaner to use
templates. Whereas before I needed 3 or 4 paragraphs to explain what `<<:` was
doing, here you see "extends" and already have an idea.

When we say "extends" here, what we're really saying is "give me everything
from this job". So in `release to testpypi`, even though I didn't say it there
it is _as if_ I had defined `stage: deploy` and `when: manual`. Note, I still
used an anchor inside of my template and I could even have my template extend
from somewhere else.

Though I didn't include it in this repository, I could have set up each of
these stages to be in their own files called something like `.build.yml`,
`.test.yml`, etc. Then I could have _included_ those files in my
`.gitlab-ci.yml` pipeline and used them as if they were defined at the top. In
that situation, however, I couldn't have used my `cache` anchor so if I wanted
to share that, I could have broken that out into its own job and had each of
these other jobs extend from cache. In this small example, that's not really
worth it. But if I had a huge project with a really big pipeline, then I might
consider refactoring my pipeline in that way.

Additionally, if I was working in an organization, I probably won't want to
write a pipeline for every project that is more or less the same. I could make
a repo that just holds pipeline templates and include templates from that repo
instead! Saves time and ensures that if I ever find a bug in one, I can patch
all of my pipelines at once.

Now with our releases to TestPyPI and PyPI defined, we are ready to deploy! But
hang on, let's say that we have something to deploy that might break the
version of our code that our users are already using? Generally speaking, it's
a bad idea for pipelines to push all the way to production without having some
sort of user input so we have also defined the `when` value in `.release` to be
"manual". This will run all of the jobs of the pipeline except for this one,
then wait for an authorized user's input before continuing.

## Invoking the Pipeline

So when does the above pipeline get invoked?  By default, it runs whenever new
changes are pushed to the repository.  If you have not already, try committing
and pushing the changes you made to setup.cfg a while back.

Stage the file:
```
git add setup.cfg
```
Then commit the file:
```
git commit -m "Changed name of package"
```
Then push the file:
```
git push
```

Now, go to your GitLab repository page and click on CI/CD > Pipelines:
![CI/CD Pipelines](ci_cd_pipelines.png)

You should a pipeline in the `running` or `passed` state for your commit.  The
four circles in the `Stages` column indicate the four stages in your pipeline.
The three check marks indicate that the first three stages (build, test,
package) have passed.  The last circle with a `>>` sign indicates that the last
deploy stage is configured to be manually triggered.  If you want to see more
details on what happened on each stage, you can click on the pipeline link.

If the pipeline is still running, wait until the first three stages have
completed and it now in passed state.  Now you are ready to trigger the manual
deploy stage.  But before you do, there is one thing you have to do.  In the
`.gitlab-ci.yml` file deploy stage, you will have noticed that four variables
have been used without being defined: TEST_PYPI_USERNAME, TEST_PYPI_PASSWORD,
PYPI_USERNAME, PYPI_PASSWORD.  These variables were intentionally omitted in
the script because it is not good practice to expose your credentials in a
public repository.  These variables can be provided to the script using the
CI/CD Settings.

On your GitLab repository page, click on Settings > CI/CD, and expand the Variables menu:
![CI/CD Settings](ci_cd_settings.png)

Now add the two variables TEST_PYPI_USERNAME and TEST_PYPI_PASSWORD as shown in
the image using the `Add variable` button.  Once you are done, you are now
ready to deploy.

Going back to the CI/CD > Pipelines menu, click on the `>>` icon representing
the last deploy stage, and then click on `release to testpypi`:
![CI/CD Deploy](ci_cd_deploy.png)

Once it is done running, you should see the `>>` icon turn into a check mark.
And if you go to https://test.pypi.org and search for the name of your package,
you should be able to see it deployed!  You should see something similar to:
https://test.pypi.org/project/simple-strop-wonsun.ahn/

In this way, every time a change is pushed, all the tests are run and a package
is created ready for upload automatically.  When you feel a need to deploy the
package, you can click on the corresponding version deploy stage.  FYI, you can
also invoke the pipeline manually if you wish by clicking on the `Run pipeline`
button.  Also, software organizations often schedule a regular invocation of
the pipeline by using the CI/CD > Schedules menu.

[1]: https://pypi.org/project/pipenv/
[2]: https://test.pypi.org/
[3]: https://packaging.python.org/tutorials/packaging-projects/
[4]: https://www.cloudbees.com/blog/yaml-tutorial-everything-you-need-get-started/
