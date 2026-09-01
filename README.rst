=====================================================================
 django-celery-beat-next
=====================================================================

A fork of `django-celery-beat`_ that you can install with Django 6.1.

|build-status| |coverage| |license| |pyversion| |pyimp|

:PyPI: django-celery-beat-next
:Import: ``django_celery_beat``
:Source: https://github.com/azataiot/django-celery-beat-next
:Upstream: https://github.com/celery/django-celery-beat
:Issue: `celery/django-celery-beat#1079`_
:Keywords: django, celery, beat, periodic task, cron, scheduling

This fork is not an official Celery project.

Why this fork exists
====================

``django-celery-beat`` 2.9.0 requires ``Django<6.1``.
A lockfile cannot take Django 6.1 while that requirement stands.

Upstream ``main`` already runs the 6.1 tests.
The requirement there is ``Django>=3.2.25,<6.2``.
See `celery/django-celery-beat#1079`_ and `celery/django-celery-beat#1042`_.

This repo follows that ``main`` branch.
The import and the Django app label stay ``django_celery_beat``,
so you do not touch migrations.
Swap the dependency name only. Leave ``INSTALLED_APPS`` as it is.

Do not depend on both packages at once. They install the same module.

If a later ``django-celery-beat`` release accepts Django 6.1,
switch to that release. Then drop this fork.

.. _django-celery-beat: https://github.com/celery/django-celery-beat
.. _celery/django-celery-beat#1079: https://github.com/celery/django-celery-beat/issues/1079
.. _celery/django-celery-beat#1042: https://github.com/celery/django-celery-beat/pull/1042

About
=====

This extension enables you to store the periodic task schedule in the
database.

The periodic tasks can be managed from the Django Admin interface, where you
can create, edit and delete periodic tasks and how often they should run.

Using the Extension
===================

Usage and installation instructions for this extension are available
from the `Celery documentation`_.

.. _`Celery documentation`:
    http://docs.celeryproject.org/en/latest/userguide/periodic-tasks.html#using-custom-scheduler-classes

Important Warning about Time Zones
==================================

.. warning::
   If you change the Django ``TIME_ZONE`` setting your periodic task schedule
   will still be based on the old timezone.

   To fix that you would have to reset the "last run time" for each periodic task:

.. code-block:: Python

        >>> from django_celery_beat.models import PeriodicTask, PeriodicTasks
        >>> PeriodicTask.objects.all().update(last_run_at=None)
        >>> PeriodicTasks.update_changed()



.. note::
   This will reset the state as if the periodic tasks have never run before.


Models
======

- ``django_celery_beat.models.PeriodicTask``

This model defines a single periodic task to be run.

It must be associated with a schedule, which defines how often the task should
run.

- ``django_celery_beat.models.IntervalSchedule``

A schedule that runs at a specific interval (e.g. every 5 seconds).

- ``django_celery_beat.models.CrontabSchedule``

A schedule with fields like entries in cron:
``minute hour day-of-week day_of_month month_of_year``.

- ``django_celery_beat.models.PeriodicTasks``

This model is only used as an index to keep track of when the schedule has
changed.

Whenever you update a ``PeriodicTask`` a counter in this table is also
incremented, which tells the ``celery beat`` service to reload the schedule
from the database.

If you update periodic tasks in bulk, you will need to update the counter
manually:

.. code-block:: Python

        >>> from django_celery_beat.models import PeriodicTasks
        >>> PeriodicTasks.update_changed()

Example creating interval-based periodic task
---------------------------------------------

To create a periodic task executing at an interval you must first
create the interval object:

.. code-block:: Python

        >>> from django_celery_beat.models import PeriodicTask, IntervalSchedule

        # executes every 10 seconds.
        >>> schedule, created = IntervalSchedule.objects.get_or_create(
        ...     every=10,
        ...     period=IntervalSchedule.SECONDS,
        ... )

That's all the fields you need: a period type and the frequency.

You can choose between a specific set of periods:


- ``IntervalSchedule.DAYS``
- ``IntervalSchedule.HOURS``
- ``IntervalSchedule.MINUTES``
- ``IntervalSchedule.SECONDS``
- ``IntervalSchedule.MICROSECONDS``

.. note::
    If you have multiple periodic tasks executing every 10 seconds,
    then they should all point to the same schedule object.

There's also a "choices tuple" available should you need to present this
to the user:


.. code-block:: Python

        >>> IntervalSchedule.PERIOD_CHOICES


Now that we have defined the schedule object, we can create the periodic task
entry:

.. code-block:: Python

        >>> PeriodicTask.objects.create(
        ...     interval=schedule,                  # we created this above.
        ...     name='Importing contacts',          # simply describes this periodic task.
        ...     task='proj.tasks.import_contacts',  # name of task.
        ... )


Note that this is a very basic example, you can also specify the arguments
and keyword arguments used to execute the task, the ``queue`` to send it
to[*], and set an expiry time.

Here's an example specifying the arguments, note how JSON serialization is
required:

.. code-block:: Python

        >>> import json
        >>> from datetime import datetime, timedelta

        >>> PeriodicTask.objects.create(
        ...     interval=schedule,                  # we created this above.
        ...     name='Importing contacts',          # simply describes this periodic task.
        ...     task='proj.tasks.import_contacts',  # name of task.
        ...     args=json.dumps(['arg1', 'arg2']),
        ...     kwargs=json.dumps({
        ...        'be_careful': True,
        ...     }),
        ...     expires=datetime.utcnow() + timedelta(seconds=30)
        ... )


.. [*] you can also use low-level AMQP routing using the ``exchange`` and
       ``routing_key`` fields.

Example creating crontab-based periodic task
--------------------------------------------

A crontab schedule has the fields: ``minute``, ``hour``, ``day_of_week``,
``day_of_month`` and ``month_of_year``, so if you want the equivalent
of a ``30 * * * *`` (execute 30 minutes past every hour) crontab entry you specify:

.. code-block:: Python

        >>> from django_celery_beat.models import CrontabSchedule, PeriodicTask
        >>> schedule, _ = CrontabSchedule.objects.get_or_create(
        ...     minute='30',
        ...     hour='*',
        ...     day_of_week='*',
        ...     day_of_month='*',
        ...     month_of_year='*',
        ...     timezone=zoneinfo.ZoneInfo('Canada/Pacific')
        ... )

The crontab schedule is linked to a specific timezone using the 'timezone' input parameter.

Then to create a periodic task using this schedule, use the same approach as
the interval-based periodic task earlier in this document, but instead
of ``interval=schedule``, specify ``crontab=schedule``:

.. code-block:: Python

        >>> PeriodicTask.objects.create(
        ...     crontab=schedule,
        ...     name='Importing contacts',
        ...     task='proj.tasks.import_contacts',
        ... )

Temporarily disable a periodic task
-----------------------------------

You can use the ``enabled`` flag to temporarily disable a periodic task:

.. code-block:: Python

        >>> periodic_task.enabled = False
        >>> periodic_task.save()


Example running periodic tasks
-----------------------------------

The periodic tasks still need 'workers' to execute them.
So make sure the default **Celery** package is installed.
(If not installed, please follow the installation instructions
here: https://github.com/celery/celery)

Both the worker and beat services need to be running at the same time.

1. Start a Celery worker service (specify your Django project name)::

   $ celery -A [project-name] worker --loglevel=info


2. As a separate process, start the beat service (specify the Django scheduler)::

    $ celery -A [project-name] beat -l info --scheduler django_celery_beat.schedulers:DatabaseScheduler

   **OR** you can use the -S (scheduler flag), for more options see ``celery beat --help``)::

    $ celery -A [project-name] beat -l info -S django

   Also, as an alternative, you can run the two steps above (worker and beat services)
   with only one command (recommended for **development environment only**)::

    $ celery -A [project-name] worker --beat --scheduler django --loglevel=info


3. Now you can add and manage your periodic tasks from the Django Admin interface.




Installation
============

You can install django-celery-beat-next either via the Python Package Index (PyPI)
or from source.

To install using ``pip``:

.. code-block:: bash

        $ pip install --upgrade django-celery-beat-next

Downloading and installing from source
--------------------------------------

Download the latest version of django-celery-beat-next from
https://pypi.org/project/django-celery-beat-next/

You can install it by doing the following :

.. code-block:: bash

        $ python3 -m venv .venv
        $ source .venv/bin/activate
        $ pip install --upgrade build pip
        $ tar xvfz django-celery-beat-next-0.0.0.tar.gz
        $ cd django-celery-beat-next-0.0.0
        $ python -m build
        $ pip install --upgrade .

After installation, add ``django_celery_beat`` to Django's settings module:


.. code-block:: Python

        INSTALLED_APPS = [
            ...,
            'django_celery_beat',
        ]


Run the ``django_celery_beat`` migrations using:

.. code-block:: bash

        $ python manage.py migrate django_celery_beat


Using the development version
-----------------------------

With pip
~~~~~~~~

You can install the latest main version of django-celery-beat-next using the following
pip command:

.. code-block:: bash

        $ pip install git+https://github.com/azataiot/django-celery-beat-next.git#egg=django-celery-beat-next


Developing django-celery-beat
-----------------------------

To spin up a local development copy of django-celery-beat with Django admin at http://127.0.0.1:58000/admin/ run:

.. code-block:: bash

        $ docker-compose up --build

Log-in as user ``admin`` with password ``admin``.


TZ Awareness:
-------------

If you have a project that is time zone naive, you can set ``DJANGO_CELERY_BEAT_TZ_AWARE=False`` in your settings file.


.. |build-status| image:: https://github.com/azataiot/django-celery-beat-next/actions/workflows/test.yml/badge.svg
    :alt: Build status
    :target: https://github.com/azataiot/django-celery-beat-next/actions/workflows/test.yml

.. |coverage| image:: https://codecov.io/github/celery/django-celery-beat/coverage.svg?branch=main
    :target: https://codecov.io/github/celery/django-celery-beat?branch=main

.. |license| image:: https://img.shields.io/pypi/l/django-celery-beat.svg
    :alt: BSD License
    :target: https://opensource.org/licenses/BSD-3-Clause

.. |pyversion| image:: https://img.shields.io/pypi/pyversions/django-celery-beat.svg
    :alt: Supported Python versions.
    :target: https://pypi.org/project/django-celery-beat-next/

.. |pyimp| image:: https://img.shields.io/pypi/implementation/django-celery-beat.svg
    :alt: Support Python implementations.
    :target: https://pypi.org/project/django-celery-beat-next/
