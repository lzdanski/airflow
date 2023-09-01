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
~~~~~~~~~~~~~

By default, Airflow waits for all upstream (direct parents) tasks for a task to be :ref:`successful <concepts:task-states>` before it runs that task.

However, this is just the default behaviour, and you can control it using the ``trigger_rule`` argument to a Task. The options for ``trigger_rule`` are:

* ``all_success`` (default): All upstream tasks have succeeded
* ``all_failed``: All upstream tasks are in a ``failed`` or ``upstream_failed`` state
* ``all_done``: All upstream tasks are done with their execution
* ``all_skipped``: All upstream tasks are in a ``skipped`` state
* ``one_failed``: At least one upstream task has failed (does not wait for all upstream tasks to be done)
* ``one_success``: At least one upstream task has succeeded (does not wait for all upstream tasks to be done)
* ``one_done``: At least one upstream task succeeded or failed
* ``none_failed``: All upstream tasks have not ``failed`` or ``upstream_failed`` - that is, all upstream tasks have succeeded or been skipped
* ``none_failed_min_one_success``: All upstream tasks have not ``failed`` or ``upstream_failed``, and at least one upstream task has succeeded.
* ``none_skipped``: No upstream task is in a ``skipped`` state - that is, all upstream tasks are in a ``success``, ``failed``, or ``upstream_failed`` state
* ``always``: No dependencies at all, run this task at any time


You can also combine this with the :ref:`concepts:depends-on-past` functionality if you wish.

.. note::

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