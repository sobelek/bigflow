# Project structure and build

## Overview

In this chapter, you will get to know the anatomy of a standard BigFlow project, deployment artifacts, and [CLI commands](./cli.md).

The BigFlow build command packages your processing code into a Docker image. Thanks to this approach, you can create any
environment you want for your workflows. What is more, you don't need to worry about dependencies clashes on the Cloud Composer (Airflow).

There are two types of artifacts which BigFlow produces:

* Deployment artifact — that's a final build product which you deploy.
* Intermediate artifact — an element of a deployment artifact, or a by-product, typically useful for debugging.

## Deployment artifacts

The concept of BigFlow deployment artifacts looks like this:

![BigFlow artifact schema](./images/bigflow-artifact.png)

BigFlow turns your project into a standard Python package (which can be uploaded to [pypi](https://pypi.org/) or
installed locally using `pip`).
Next, the package is installed on a Docker [image](#docker-image) with a fixed Python version.
Finally, BigFlow generates Airflow [DAG files](#dag) which use this image.

From each of your [workflows](./workflow-and-job.md#workflow), BigFlow generates a DAG file.
Every produced DAG consists only of [`KubernetesPodOperator`](https://airflow.apache.org/docs/stable/_api/airflow/contrib/operators/kubernetes_pod_operator/index.html) objects, which
executes operations on a Docker image.

To build a project you need to use the [`bigflow build`](./cli.md#building-airflow-dags) command. The documentation, you are reading, is also a valid BigFlow
project. Go to the [`docs`](../docs) directory and run the `bigflow build` command to see how the build process works.

The `bigflow build` command should produce:

* The `dist` directory with a **Python package** (intermediate artifact)
* The `build` directory with **JUnit test results** (intermediate artifact)
* The `.image` directory with a **deployment configuration** and **Docker image** as `.tar` (deployment artifact)
* The `.dags` directory with Airflow **DAGs**, generated from workflows (deployment artifact)

The `bigflow build` command uses three subcommands to generate all the
artifacts: [`bigflow build-package`](./cli.md#building-airflow-dags), [`bigflow build-image`](./cli.md#building-airflow-dags), [`bigflow build-dags`](./cli.md#building-airflow-dags).
There is also an optional [`bigflow build-requirements`](./cli.md#building-airflow-dags) command that allows you
to [resolve and freeze](https://github.com/jazzband/pip-tools) the project dependencies.

Now, let us go through each building element in detail, starting from the Python package.

## Package

### Project structure

An exemplary BigFlow project, with the standard structure, looks like this:

```
project_dir/
    project_package/
        __init__.py
        workflow.py
    test/
        __init__.py
    resources/
        requirements.in
        requirements.txt
    Dockerfile
    deployment_config.py
    setup.py
    pyproject.toml
```

Let us start with the `project_package`. It's the Python package which contains the processing logic of your workflows.
It also contains `Workflow` objects, which arranges parts of your processing logic into
a [DAG](https://en.wikipedia.org/wiki/Directed_acyclic_graph) (read the [Workflow & Job chapter](./workflow-and-job.md) to learn more about workflows and jobs).
The `project_package` is used to create a standard Python package, which can be installed using `pip`.

`setup.py` is the build script for the project. It turns the `project_package` into a `.whl` package.
It's based on the standard Python tool — [setuptool](https://packaging.python.org/key_projects/#setuptools).

You can put your tests into the `test` package. The `bigflow build-package` command runs tests automatically, before trying to build the package.

The `resources` directory contains non-Python files. That is the only directory that will be packaged
along the `project_package` (so you can access these files after installation, from the `project_package`). Any other files
inside the `project_directory` won't be available in the `project_package`. `resources` can't be nested. So you can't
have a directory inside the `resources` directory.

The `get_resource_absolute_path` function allows you to access files from the `resources` directory.

[**`resources.py`**](examples/project_structure_and_build/resources_workflow.py)
```python
with open(get_resource_absolute_path('example_resource.txt', Path(__file__))) as f:
    print(f.read())
```

Run the above example, using the following command:

```shell script
bigflow run --workflow resources_workflow
```

Result:

```
Welcome inside the example resource!
```

Because every BigFlow project is a standard Python package, we suggest going through the
[official Python packaging tutorial](https://packaging.python.org/tutorials/packaging-projects/).

### Package builder

The `bigflow build-package` command takes three steps to build a Python package from your project:

1. Cleans leftovers from a previous build.
1. Runs tests for the project. You can find the generated report inside the `build/junit-reports` directory.
1. Runs the `bdist_wheel` setup tools command. It generates a `.whl` package which you can
upload to `pypi` or install locally - `pip install your_generated_package.whl`.

Go to the [`docs`](../docs) project and run the `bigflow build-package` command to observe the result. Now you can install the
generated package using `pip install dist/examples-0.1.0-py3-none-any.whl`. After you install the `.whl` file, you can
run jobs and workflows from the `docs/examples` package. They are now installed in your virtual environment, so you can run them
anywhere in your directory tree, for example:

```shell script
cd /tmp
bigflow run --workflow hello_world_workflow --project-package examples
```

You probably won't use this command very often, but it's useful for debugging. Sometimes you want to see if your project
works as you expect in the form of a package (and not just as a package in your project directory).

### Project versioning

Deployment artifacts, like Docker images, need to be versioned. BigFlow provides automatic versioning based on
the git tags system. There are two commands you need to know.

The `bigflow project-version` command prints the current version of your project:

```
bigflow project-version
>>> 0.34.0
```

BigFlow follows the standard [semver](https://en.wikipedia.org/wiki/Software_versioning) schema:

`<major>.<minor>.<patch>`

If BigFlow finds a tag on a current commit, it uses it as a current project version.
If there are commits after the last tag or a working directory is dirty, it creates a snapshot version with the
following schema:

`<major>.<minor>.<patch><snapshot_id>`

For example:

```
bigflow project-version
>>> 0.34.0SHAdee9af83SNAPSHOT8650450a
```

If you are ready to release a new version, you don't have to set a new tag manually. You can use the `bigflow release` command:

```
bigflow project-version
>>> 0.34.0SHAdee9af83SNAPSHOT8650450a
bigflow release
bigflow project-version
>>> 0.35.0
```

If needed, you can specify an identity file for ssh, used to push a tag to a remote repository.

```
bigflow release --ssh-identity-file /path/to/id_rsa
bigflow release -i keys.pem

```

## Test framework

Bigflow automatically runs tests via [`unittest`](https://docs.python.org/3/library/unittest.html) or [`pytest`](https://docs.pytest.org/) framework:

* `unittest` - deafult testing framework for Python, has similar api to Java JUnit.
* `pytest` - "pythonic" framework with lighter API, can also run tests written for `unittest` .

By deafult `unittest` is used, no extra configuration is needed.  Switching to `pytest` may be done by adding  `pytest` into `requirements.in` and specifying `test_framework` option in `setup.py`:

```python
import bigflow.build
bigflow.build.setup(
    ...
    test_framework='pytest',  # either 'unittest' or 'pytest'
)
```

## Docker image

To run a job in a desired environment, BigFlow makes use of Docker. Each job is executed from a Docker container,
which runs a Docker image built from your project. The default [`Dockerfile`](https://docs.docker.com/engine/reference/builder/)
generated from the scaffolding tool looks like this:

[**`Dockerfile`**](Dockerfile)

```dockerfile
FROM python:3.7
COPY ./dist /dist
RUN apt-get -y update && apt-get install -y libzbar-dev libc-dev musl-dev
RUN pip install pip==20.2.4
RUN for i in /dist/*.whl; do pip install $i --use-feature=2020-resolver; done
```

The basic image installs the generated Python package. With the installed package, you can run a workflow or a job
from a Docker environment.

Run the `bigflow build-image` command inside the [`docs`](../docs) project. Next, you can run the example workflow using Docker:

```shell script
docker load -i .image/image-{Generated image version}.tar
docker run {Your loaded image ID} bigflow run --job hello_world_workflow.hello_world --project-package examples
```

DAGs generated by BigFlow use KubernetesPodOperator to call this Docker command.

## DAG

BigFlow generates Airflow DAGs from workflows found in your project.

Every generated DAG utilizes only [`KubernetesPodOperator`](https://airflow.apache.org/docs/stable/_api/airflow/contrib/operators/kubernetes_pod_operator/index.html).

To see how it works, go to the [`docs`](../docs) project and run the `bigflow build-dags` command.

One of the generated DAGs, for the [`resources.py`](examples/project_structure_and_build/resources_workflow.py) workflow, looks like this:

```python
import datetime
from airflow import DAG
from airflow.contrib.operators import kubernetes_pod_operator

default_args = {
            'owner': 'airflow',
            'depends_on_past': True,
            'start_date': datetime.datetime(2020, 8, 30),
            'email_on_failure': False,
            'email_on_retry': False,
            'execution_timeout': datetime.timedelta(minutes=180),
}

dag = DAG(
    'resources_workflow__v0_1_0__2020_08_31_15_00_00',
    default_args=default_args,
    max_active_runs=1,
    schedule_interval='@daily'
)


print_resource_job = kubernetes_pod_operator.KubernetesPodOperator(
    task_id='print-resource-job',
    name='print-resource-job',
    cmds=['bf'],
    arguments=['run', '--job', 'resources_workflow.print_resource_job', '--runtime', '{{ execution_date.strftime("%Y-%m-%d %H:%M:%S") }}', '--project-package', 'examples', '--config', '{{var.value.env}}'],
    namespace='default',
    image='eu.gcr.io/docker_repository_project/my-project:0.1.0',
    is_delete_operator_pod=True,
    retries=3,
    retry_delay=timedelta(seconds=60),
    dag=dag)
```

Every [job](./workflow-and-job.md#job) in a [workflow](./workflow-and-job.md#workflow) maps to
[`KubernetesPodOperator`](https://airflow.apache.org/docs/stable/_api/airflow/contrib/operators/kubernetes_pod_operator/index.html).
BigFlow sets a reasonable default value for the required operator arguments. You can modify
[some of them](./workflow-and-job.md#job), by setting properties on a job.
Similarly, you can modify a DAG property, by setting [properties on a workflow](./workflow-and-job.md#workflow-scheduling-options).

## requirements.in

In the `resources` directory you can find `requirements.txt` and `requirements.in` files. You should keep your project
dependencies in the `requirements.in` file. Then, you can resolve and freeze your project requirements into the `requirements.txt`,
using the `bigflow build-requirements` command.

Under the hood, the `build-requirements` command uses the [`pip-tools`](https://github.com/jazzband/pip-tools).

The `build-requirements` command is part of the `build` command, but it's not mandatory to use `requirements.in`. That
mechanism is optional, so you can just use `requirements.txt` alone. BigFlow automatically detects if you have `requirements.in`
in the `resources` directory and generates or updates the `requirements.txt`. If `requirements.in` is not there, then BigFlow
just skips the `build-requirements` phase.

Using `requirements.in` is the recommended way of managing dependencies. It is because it makes your artifacts deterministic (at least
when it comes to resolving dependencies). If you don't use frozen requirements, then each time you install requirements, you
can get a different result. In theory it shouldn't be a problem, because you should get compatible dependencies, but in practice
you might get a incompatible dependency which breaks your processing.

## pyproject.toml

The `pyproject.toml` file is [part of the standard Python packaging toolset](https://snarky.ca/what-the-heck-is-pyproject-toml/).
BigFlow uses that file to describe requirements needed to build the project. That description is especially useful in 2 situations:

* Building the project using systems like Bamboo, Jenkins, Travis, etc.
* Running Apache Beam jobs (go to the [chapter about Beam inside BigFlow](technologies.md#dataflow-apache-beam) to understand why).