 .. Licensed to the Apache Software Foundation (ASF) under one
    or more contributor license agreements.  See the NOTICE file
    distributed with this work for additional information
    regarding copyright ownership.  The ASF licenses this file
    to you under the Apache License, Version 2.0 (the
    "License"); you may not use this file except in compliance
    with the License.  You may obtain a copy of the License at

 ..   http://www.apache.org/licenses/LICENSE-2.0

 .. Unless required by applicable law or agreed to in writing,
    software distributed under the License is distributed on an
    "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
    KIND, either express or implied.  See the License for the
    specific language governing permissions and limitations
    under the License.




Fundamental Concepts
====================

This tutorial walks you through some of the fundamental Airflow concepts,
objects, and their usage while writing your first DAG.

Example Pipeline definition
---------------------------

Here is an example of a basic pipeline definition. Do not worry if this looks
complicated, a line by line explanation follows below.

.. exampleinclude:: /../../airflow-core/src/airflow/example_dags/tutorial.py
    :language: python
    :start-after: [START tutorial]
    :end-before: [END tutorial]

It's a DAG definition file
--------------------------

One thing to wrap your head around (it may not be very intuitive for everyone
at first) is that this Airflow Python script is really
just a configuration file specifying the DAG's structure as code.
The actual tasks defined here will run in a different context from
the context of this script. Different tasks run on different workers
at different points in time, which means that this script cannot be used
to cross communicate between tasks. Note that for this
purpose we have a more advanced feature called :doc:`/core-concepts/xcoms`.

People sometimes think of the DAG definition file as a place where they
can do some actual data processing - that is not the case at all!
The script's purpose is to define a DAG object. It needs to evaluate
quickly (seconds, not minutes) since the scheduler will execute it
periodically to reflect the changes if any.


Importing Modules
-----------------

An Airflow pipeline is just a Python script that happens to define an
Airflow DAG object. Let's start by importing the libraries we will need.

.. exampleinclude:: /../../airflow-core/src/airflow/example_dags/tutorial.py
    :language: python
    :start-after: [START import_module]
    :end-before: [END import_module]


See :doc:`/administration-and-deployment/modules_management` for details on how Python and Airflow manage modules.

Default Arguments
-----------------
We're about to create a DAG and some tasks, and we have the choice to
explicitly pass a set of arguments to each task's constructor
(which would become redundant), or (better!) we can define a dictionary
of default parameters that we can use when creating tasks.

.. exampleinclude:: /../../airflow-core/src/airflow/example_dags/tutorial.py
    :language: python
    :dedent: 4
    :start-after: [START default_args]
    :end-before: [END default_args]

For more information about the BaseOperator's parameters and what they do,
refer to the :py:class:`airflow.models.baseoperator.BaseOperator` documentation.

Also, note that you could easily define different sets of arguments that
would serve different purposes. An example of that would be to have
different settings between a production and development environment.


Instantiate a DAG
-----------------

We'll need a DAG object to nest our tasks into. Here we pass a string
that defines the ``dag_id``, which serves as a unique identifier for your DAG.
We also pass the default argument dictionary that we just defined and
define a ``schedule`` of 1 day for the DAG.

.. exampleinclude:: /../../airflow-core/src/airflow/example_dags/tutorial.py
    :language: python
    :start-after: [START instantiate_dag]
    :end-before: [END instantiate_dag]

Operators
---------

An operator defines a unit of work for Airflow to complete. Using operators is the classic approach
to defining work in Airflow. For some use cases, it's better to use the TaskFlow API to define
work in a Pythonic context as described in :doc:`taskflow`. For now, using operators helps to
visualize task dependencies in our DAG code.

All operators inherit from the BaseOperator, which includes all of the required arguments for
running work in Airflow. From here, each operator includes unique arguments for
the type of work it's completing. Some of the most popular operators are the PythonOperator, the BashOperator, and the
KubernetesPodOperator.

Airflow completes work based on the arguments you pass to your operators. In this tutorial, we
use the BashOperator to run a few bash scripts.

Tasks
-----

To use an operator in a DAG, you have to instantiate it as a task. Tasks
determine how to execute your operator's work within the context of a DAG.

In the following example, we instantiate the BashOperator as two separate tasks in order to run two
separate bash scripts. The first argument for each instantiation, ``task_id``,
acts as a unique identifier for the task.

.. exampleinclude:: /../../airflow-core/src/airflow/example_dags/tutorial.py
    :language: python
    :dedent: 4
    :start-after: [START basic_task]
    :end-before: [END basic_task]

Notice how we pass a mix of operator specific arguments (``bash_command``) and
an argument common to all operators (``retries``) inherited
from BaseOperator to the operator's constructor. This is simpler than
passing every argument for every constructor call. Also, notice that in
the second task we override the ``retries`` parameter with ``3``.

The precedence rules for a task are as follows:

1.  Explicitly passed arguments
2.  Values that exist in the ``default_args`` dictionary
3.  The operator's default value, if one exists

.. note::
    A task must include or inherit the arguments ``task_id`` and ``owner``,
    otherwise Airflow will raise an exception. A fresh install of Airflow will
    have a default value of 'airflow' set for ``owner``, so you only really need
    to worry about ensuring ``task_id`` has a value.

Templating with Jinja
---------------------
Airflow leverages the power of
`Jinja Templating <https://jinja.palletsprojects.com/en/2.11.x/>`_ and provides
the pipeline author
with a set of built-in parameters and macros. Airflow also provides
hooks for the pipeline author to define their own parameters, macros and
templates.

This tutorial barely scratches the surface of what you can do with
templating in Airflow, but the goal of this section is to let you know
this feature exists, get you familiar with double curly brackets, and
point to the most common template variable: ``{{ ds }}`` (today's "date
stamp").

.. exampleinclude:: /../../airflow-core/src/airflow/example_dags/tutorial.py
    :language: python
    :dedent: 4
    :start-after: [START jinja_template]
    :end-before: [END jinja_template]

Notice that the ``templated_command`` contains code logic in ``{% %}`` blocks,
references parameters like ``{{ ds }}``, and calls a function as in
``{{ macros.ds_add(ds, 7)}}``.

Files can also be passed to the ``bash_command`` argument, like
``bash_command='templated_command.sh'``, where the file location is relative to
the directory containing the pipeline file (``tutorial.py`` in this case). This
may be desirable for many reasons, like separating your script's logic and
pipeline code, allowing for proper code highlighting in files composed in
different languages, and general flexibility in structuring pipelines. It is
also possible to define your ``template_searchpath`` as pointing to any folder
locations in the DAG constructor call.

Using that same DAG constructor call, it is possible to define
``user_defined_macros`` which allow you to specify your own variables.
For example, passing ``dict(foo='bar')`` to this argument allows you
to use ``{{ foo }}`` in your templates. Moreover, specifying
``user_defined_filters`` allows you to register your own filters. For example,
passing ``dict(hello=lambda name: 'Hello %s' % name)`` to this argument allows
you to use ``{{ 'world' | hello }}`` in your templates. For more information
regarding custom filters have a look at the
`Jinja Documentation <https://jinja.palletsprojects.com/en/latest/api/#custom-filters>`_.

For more information on the variables and macros that can be referenced
in templates, make sure to read through the :ref:`templates-ref`.

Adding DAG and Tasks documentation
----------------------------------
We can add documentation for DAG or each single task. DAG documentation only supports
markdown so far, while task documentation supports plain text, markdown, reStructuredText,
json, and yaml. The DAG documentation can be written as a doc string at the beginning
of the DAG file (recommended), or anywhere else in the file. Below you can find some examples
on how to implement task and DAG docs, as well as screenshots:

.. exampleinclude:: /../../airflow-core/src/airflow/example_dags/tutorial.py
    :language: python
    :dedent: 4
    :start-after: [START documentation]
    :end-before: [END documentation]

.. image:: ../img/task_doc.png
.. image:: ../img/dag_doc.png

Setting up Dependencies
-----------------------
We have tasks ``t1``, ``t2`` and ``t3`` that depend on each other. Here's a few ways
you can define dependencies between them:

.. code-block:: python

    t1.set_downstream(t2)

    # This means that t2 will depend on t1
    # running successfully to run.
    # It is equivalent to:
    t2.set_upstream(t1)

    # The bit shift operator can also be
    # used to chain operations:
    t1 >> t2

    # And the upstream dependency with the
    # bit shift operator:
    t2 << t1

    # Chaining multiple dependencies becomes
    # concise with the bit shift operator:
    t1 >> t2 >> t3

    # A list of tasks can also be set as
    # dependencies. These operations
    # all have the same effect:
    t1.set_downstream([t2, t3])
    t1 >> [t2, t3]
    [t2, t3] << t1

Note that when executing your script, Airflow will raise exceptions when
it finds cycles in your DAG or when a dependency is referenced more
than once.

Using time zones
----------------

Creating a time zone aware DAG is quite simple. Just make sure to supply a time zone aware dates
using ``pendulum``. Don't try to use standard library
`timezone <https://docs.python.org/3/library/datetime.html#timezone-objects>`_ as they are known to
have limitations and we deliberately disallow using them in dags.

Recap
-----
Alright, so we have a pretty basic DAG. At this point your code should look
something like this:

.. exampleinclude:: /../../airflow-core/src/airflow/example_dags/tutorial.py
    :language: python
    :start-after: [START tutorial]
    :end-before: [END tutorial]

.. _testing:

Testing
--------

Running the Script
''''''''''''''''''

Time to run some tests. First, let's make sure the pipeline
is parsed successfully.

Let's assume we are saving the code from the previous step in
``tutorial.py`` in the dags folder referenced in your ``airflow.cfg``.
The default location for your dags is ``~/airflow/dags``.

.. code-block:: bash

    python ~/airflow/dags/tutorial.py

If the script does not raise an exception it means that you have not done
anything horribly wrong, and that your Airflow environment is somewhat
sound.

Command Line Metadata Validation
'''''''''''''''''''''''''''''''''
Let's run a few commands to validate this script further.

.. code-block:: bash

    # initialize the database tables
    airflow db migrate

    # print the list of active dags
    airflow dags list

    # prints the list of tasks in the "tutorial" DAG
    airflow tasks list tutorial

    # prints the hierarchy of tasks in the "tutorial" DAG
    airflow tasks list tutorial --tree


Testing
'''''''
Let's test by running the actual task instances for a specific date. The date
specified in this context is called the *logical date* (also called *execution
date* for historical reasons), which simulates the scheduler running your task
or DAG for a specific date and time, even though it *physically* will run now
(or as soon as its dependencies are met).

We said the scheduler runs your task *for* a specific date and time, not *at*.
This is because each run of a DAG conceptually represents not a specific date
and time, but an interval between two times, called a
:ref:`data interval <data-interval>`. A DAG run's logical date is the start of
its data interval.

.. code-block:: bash

    # command layout: command subcommand [dag_id] [task_id] [(optional) date]

    # testing print_date
    airflow tasks test tutorial print_date 2015-06-01

    # testing sleep
    airflow tasks test tutorial sleep 2015-06-01

Now remember what we did with templating earlier? See how this template
gets rendered and executed by running this command:

.. code-block:: bash

    # testing templated
    airflow tasks test tutorial templated 2015-06-01

This should result in displaying a verbose log of events and ultimately
running your bash command and printing the result.

Note that the ``airflow tasks test`` command runs task instances locally, outputs
their log to stdout (on screen), does not bother with dependencies, and
does not communicate state (running, success, failed, ...) to the database.
It simply allows testing a single task instance.

The same applies to ``airflow dags test``, but on a DAG
level. It performs a single DAG run of the given DAG id. While it does take task
dependencies into account, no state is registered in the database. It is
convenient for locally testing a full run of your DAG, given that e.g. if one of
your tasks expects data at some location, it is available.

What's Next?
-------------
That's it! You have written and tested your very first Airflow
pipeline. Merging your code into a repository that has a Scheduler
running against it should result in being triggered and run every day.

Here are a few things you might want to do next:

.. seealso::
    - Continue to the next step of the tutorial: :doc:`/tutorial/taskflow`
    - Skip to the :doc:`/core-concepts/index` section for detailed explanation of Airflow concepts such as dags, Tasks, Operators, and more
