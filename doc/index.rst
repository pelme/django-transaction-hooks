Welcome to django-transaction-hooks!
====================================

A better alternative to the transaction signals Django `will never have`_.

Sometimes you need to fire off an action related to the current database
transaction, but only if the transaction successfully commits. Examples: a
`Celery`_ task, an email notification, or a cache invalidation.

Doing this correctly while accounting for savepoints that might be individually
rolled back, closed/dropped connections, and idiosyncrasies of various
databases, `is non-trivial`_. Transaction signals just make it easier to do it
wrong.

``django-transaction-hooks`` does the heavy lifting so you don't have to.

.. _will never have: https://code.djangoproject.com/ticket/14051
.. _Celery: http://www.celeryproject.org/
.. _is non-trivial: https://github.com/aaugustin/django-transaction-signals


Prerequisites
-------------

``django-transaction-hooks`` supports `Django`_ 1.6.1 and later on Python 2.6,
2.7, 3.2, and 3.3.

SQLite3, PostgreSQL, and MySQL are currently the only databases with built-in
support; you can experiment with whether it works for your favorite database
backend with just `a few lines of code`_.

.. _Django: http://www.djangoproject.com/


Installation
------------

``django-transaction-hooks`` is available on `PyPI`_. Install it with::

    pip install django-transaction-hooks

.. _PyPI: https://pypi.python.org/pypi/django-transaction-hooks/


Setup
-----

``django-transaction-hooks`` is implemented via custom database backends. (`Why
backends?`_)

For example, to use the PosgreSQL backend, set the ``ENGINE`` in your
``DATABASES`` setting to ``transaction_hooks.backends.postgresql_psycopg2`` (in
place of ``django.db.backends.postgresql_psycopg2``). For example::

    DATABASES = {
        'default': {
            'ENGINE': 'transaction_hooks.backends.postgresql_psycopg2',
            'NAME': 'foo',
            },
        }

MySQL and SQLite are similarly supported, via
``transaction_hooks.backends.mysql`` and
``transaction_hooks.backends.sqlite3``.

.. _a few lines of code:
.. _the mixin:

Using the mixin
~~~~~~~~~~~~~~~

If you're currently using Django's built-in database backend for SQLite,
Postgres, or MySQL, you can skip this section; just use the appropriate backend
from ``transaction_hooks.backends`` as outlined above.

Not using one of those? No worries - all the magic happens in a mixin, so
making it happen with your favorite database backend may not be hard (no
guarantees it'll work right, though.)

You'll need to create your own custom backend that inherits both from
``transaction_hooks.mixin.TransactionHooksDatabaseWrapperMixin`` and from the
database backend you're currently using. To do this, make a Python package (a
directory with an ``__init__.py`` file in it) somewhere, and then put a
``base.py`` module inside that package. Its contents should look something like
this::

    from django.db.backends.postgresql_psycopg2 import base
    from transaction_hooks.mixin import TransactionHooksDatabaseWrapperMixin

    class DatabaseWrapper(TransactionHooksDatabaseWrapperMixin,
                          base.DatabaseWrapper):
        pass

Obviously you'll want to replace ``django.db.backends.postgresql_psycopg2``
with whatever existing backend you are currently using.

Then set your database ``ENGINE`` (as above) to the Python dotted path to the
package containing that ``base.py`` module. For example, if you put the above
code in ``myproject/mybackend/base.py``, your ``ENGINE`` setting would be
``myproject.mybackend``.


Usage
-----

Pass any function (that takes no arguments) to ``connection.on_commit``::

    from django.db import connection

    def do_something():
        # send a mail, fire off a Celery task, what-have-you.

    connection.on_commit(do_something)

You can also wrap your thing up in a lambda::

    connection.on_commit(lambda: some_celery_task.delay('arg1'))

The function you pass in will be called immediately after a hypothetical
database write made at the same point in your code is successfully
committed. If that hypothetical database write is instead rolled back, your
function will be discarded and never called.

If you register a callback while there is no transaction active, it will be
executed immediately.


Notes
~~~~~

.. warning::

   This code is new, not yet battle-tested, and probably has bugs. If you find
   one, please `report it`_.

.. _report it: https://github.com/carljm/django-transaction-hooks/blob/master/CONTRIBUTING.rst


Order of execution
''''''''''''''''''

On-commit hooks for a given transaction are executed in the order they were
registered.


Exception handling
''''''''''''''''''

If one on-commit hook within a given transaction raises an uncaught
exception, no later-registered hooks in that same transaction will run. (This
is, of course, the same behavior as if you'd executed the hooks sequentially
yourself without ``on_commit()``.)


Use with South
''''''''''''''

If you use `South`_, you will probably need to set the
`SOUTH_DATABASE_ADAPTERS`_ setting when you switch to a custom database backend
(e.g. to ``{'default': 'south.db.postgresql_psycopg2'}``, if you are using
PostgreSQL).

.. _South: http://south.readthedocs.org
.. _SOUTH_DATABASE_ADAPTERS: http://south.readthedocs.org/en/latest/settings.html#south-database-adapters


Use in tests
''''''''''''

Django's `TestCase`_ class wraps each test in a transaction and rolls back that
transaction after each test, in order to provide test isolation. This means
that no transaction is ever actually committed, thus your ``on_commit`` hooks
will never be run. If you need to test the results of an ``on_commit`` hook,
you may need to use `TransactionTestCase`_ instead.

.. _TestCase: https://docs.djangoproject.com/en/dev/topics/testing/tools/#django.test.TestCase
.. _TransactionTestCase: https://docs.djangoproject.com/en/dev/topics/testing/tools/#transactiontestcase


Savepoints
''''''''''

Savepoints (i.e. nested ``transaction.atomic`` blocks) are handled
correctly. That is, an ``on_commit`` hook registered after a savepoint (in a
nested ``atomic`` block) will be called after the outer transaction is
committed, but not if a rollback to that savepoint or any previous savepoint
occurred during the transaction.


Timing of execution
'''''''''''''''''''

Your hook functions are executed *after* a successful commit, so if they fail,
it will not cause the transaction to roll back. They are executed conditionally
upon the success of the transaction, but they are not *part* of the
transaction. For the intended use cases (mail notifications, Celery tasks,
etc), this is probably fine. If it's not (if your follow-up action is so
critical that its failure should mean the failure of the transaction itself),
then you don't want ``django-transaction-hooks``. (Instead, you may want to
trigger the action via a database write and thus make it properly part of the
transaction, or you may want `two-phase commit`_.)

.. _two-phase commit: http://en.wikipedia.org/wiki/Two-phase_commit_protocol


.. _why backends?:

Why database backends?
''''''''''''''''''''''

Yeah, it's a bit of a pain. But since all transaction state is stored on the
database connection object, this is the only way it can be done without
monkeypatching. And I hate monkeypatching.

(The worst bit about a custom database backend is that if you need two
different ones, they can be hard or impossible to compose together. In this
case, `the mixin`_ should make that less painful.)

If this turns out to be really popular, it might be possible to get something
like it into the Django core backends, which would remove that issue entirely.


Contributing
------------

See the `contributing docs`_.

.. _contributing docs: https://github.com/carljm/django-transaction-hooks/blob/master/CONTRIBUTING.rst
