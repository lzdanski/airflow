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

Trigger Rules
=============

By default, Airflow waits for all upstream (direct parent) tasks for a task to be :ref:`successful <concepts:task-states>` before it runs that task.

However, this is the default behaviour, and you can control it adding the ``trigger_rule`` argument to a Task. 

You can also combine this with the :ref:`concepts:depends-on-past` functionality if you wish.

Setting trigger rules
---------------------

You can assign trigger rules to both task flows and operators with the ``trigger_rule`` argument. 

The following example shows how you can set a task to print ``hi`` when all upstream tasks have completed, even if they fail or skip. 

.. code-block:: python

     @task(
    trigger_rule="all_done"
    )
    def my_task():
        return "hi"

Setting trigger rules for operators works in a similar way. The following example shows the same trigger rule in the ``BashOperator``, which configures ``my_task`` to begin when all upstream tasks complete.

.. code-block:: python

    my_task = BashOperator(
    task_id="my_task",
	bash_command="echo hi",
    trigger_rule="all_done"
    )

Trigger rule options
--------------------

The options for ``trigger_rule`` are: 

``all_success``
^^^^^^^^^^^^^^^
By default, Airflow triggers the next task in a DAG after all upstream tasks have succeeded, and are in the ``success`` state. 

    .. note::
        
        This means that if any of the parent tasks do not ``succeed``, then the task is skipped.

``all_failed``
^^^^^^^^^^^^^^
The task is triggered when all upstream tasks fail, with either a ``failed`` or ``upstream_failed`` state.

``all_done``
^^^^^^^^^^^^
Airflow triggers the next task when all upstream tasks have completed their execution, no matter the final state. 
This means upstream tasks can have a ``success``, ``failed``, or ``skipped`` state. 

``all_skipped``
^^^^^^^^^^^^^^^
Triggers a task when all upstream tasks were ``skipped``. 

``one_failed``
^^^^^^^^^^^^^^
When one upstream tasks fails, it triggers the next task.

``one_success``
^^^^^^^^^^^^^^^
As soon as one task succeeds, the task is triggered.

``one_done``
^^^^^^^^^^^^
Airflow triggers the next task as soon as one upstream task completes, regardless of the final state. This means the upstream task can have a ``success`` or ``failed``. 

``none_failed``
^^^^^^^^^^^^^^^
If all tasks upstream have succeed or are skipped, it triggers this task. 

``none_failed_min_one_success``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
If at least one upstream task succeeds, and the rest either succeed or are skipped, the task is triggered.

``none_skipped``
^^^^^^^^^^^^^^^^
Airflow triggers your task if none of the upstream tasks are ``skipped``, which means all upstream tasks are in a ``success``, ``failed``, or ``upstream_failed`` state.

``always``
^^^^^^^^^^^
There are no dependencies. The task can run at anytime.


Skipped tasks and trigger rules
"""""""""""""""""""""""""""""""

It's important to be aware of the interaction between trigger rules and skipped tasks, especially tasks that are skipped as part of a branching operation. *You almost never want to use all_success or all_failed downstream of a branching operation*.

Skipped tasks will cascade through trigger rules ``all_success`` and ``all_failed``, and cause them to skip as well. Consider the following DAG:

.. code-block:: python

    # dags/branch_without_trigger.py
    import pendulum

    from airflow.decorators import task
    from airflow.models import DAG
    from airflow.operators.empty import EmptyOperator

    dag = DAG(
        dag_id="branch_without_trigger",
        schedule="@once",
        start_date=pendulum.datetime(2019, 2, 28, tz="UTC"),
    )

    run_this_first = EmptyOperator(task_id="run_this_first", dag=dag)


    @task.branch(task_id="branching")
    def do_branching():
        return "branch_a"


    branching = do_branching()

    branch_a = EmptyOperator(task_id="branch_a", dag=dag)
    follow_branch_a = EmptyOperator(task_id="follow_branch_a", dag=dag)

    branch_false = EmptyOperator(task_id="branch_false", dag=dag)

    join = EmptyOperator(task_id="join", dag=dag)

    run_this_first >> branching
    branching >> branch_a >> follow_branch_a >> join
    branching >> branch_false >> join

``join`` is downstream of ``follow_branch_a`` and ``branch_false``. The ``join`` task will show up as skipped because its ``trigger_rule`` is set to ``all_success`` by default, and the skip caused by the branching operation cascades down to skip a task marked as ``all_success``.

.. image:: /img/branch_without_trigger.png

By setting ``trigger_rule`` to ``none_failed_min_one_success`` in the ``join`` task, we can instead get the intended behaviour:

.. image:: /img/branch_with_trigger.png