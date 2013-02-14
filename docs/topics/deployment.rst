Deploying Robust Applications
=================================

The main thing is: stop using runserver and switch over to Apache or your real server of choice.

For all deployments:

* errors: -> send out an email/sms (logtracker app - no sms integration here). http://github.com/tostan/rapidsms-tostan/tree/master/apps/logtracker/
* route startup on system boot -> route init
* route dies -> log + restart system (rapidsms-route.cron - checks pid file and process list. restarts process if dead. depends on 2, above)
* website availability from the external world -> uptime
* change your db config to be utf8-friendly. if using mysql, use InnoDB.
* make sure your server has an accurate time and date

When running on a physical machine:

* power out -> send out an email/sms (matt's childcount solution. monitors ups signals and sends out sms via a presumably-working route). in the childcount repo somewhere, i don't use this.
* remote access -> reverse ssh (matt's solution for childcount. the folks who did inveneo's amat - which is how i sneak in the backend when tostan's public url breaks, which it does a lot - agree that reverse ssh should work fine now. it wasn't around when amat was born.) i don't currently use this, am using the old amat from inveneo.

When running on a physical machine with a hardware modem:

* route hangs -> log + restart system (this is what i wrote. checks http responsiveness + sends an email. no other backend support, no outgoing sms yet. depends on this backend.)

Miscellaneously good
=======================

* install ntp to keep system clock up to date::

    apt-get install ntp
    /etc/init.d ntp stop
    ntpdate -u pool.ntp.org
    /etc/init.d ntp start

Managing long-running processes on Linux
========================================

TODO: Describe how to control and monitor long-running processes like nginx, celery, etc.

Deploying celery for production
===============================

Many of the decisions we made
:doc:`earlier </topics/celery>`
about how to install and run celery
were convenient for development, but will not work well enough for real
day-to-day use.  Here are the changes you'll need to make for production.

Broker
------

Celery uses a broker to queue up tasks and pass them to the workers. We
were using a simple broker that just runs in Django, but that broker is
not intended for production use. We need something more stable and scalable.

The most widely used broker is `RabbitMQ`_. It's
very stable, very scalable, and available for Linux, Windows, and Mac OS X.

If you're using Ubuntu or Debian, you should be able to just run:

.. code-block:: bash

    $ sudo apt-get install rabbitmq-server

and get a working version. If you want the latest, greatest, look at the
`Installing on Debian`_ page.


Otherwise, go to the
`RabbitMQ download page`_ and follow the instructions there to install the
RabbitMQ server. You don't need the client, as Celery comes with that support
built-in.

You'll only need one RabbitMQ server instance running, even for very large
sites.

The only configuration you'll likely need to do is to delete the default,
``guest``, user and then set up a new user.  For this example, we'll
name the new user ``rabbituser``; you can use any name you want.
(This is not a user on the server itself, it's internal to RabbitMQ.)

.. code-block:: bash

    $ sudo rabbitmqctl delete_user guest
    $ sudo rabbitmqctl add_user rabbituser password
    $ sudo rabbitmqctl set_permissions rabbituser '.*' '.*' '.*'
    # This next line might not work on versions before 3.0; if so, just ignore
    $ sudo rabbitmqctl set_user_tags rabbituser administrator

Now, you'll need to make a few changes to your Django settings.

* Remove the old ``BROKER_URL`` setting.

* Remove ``kombu.transport.django`` from ``INSTALLED_APPS``.

* Create a new ``BROKER_URL`` setting that tells Celery how to connect
  to your RabbitMQ server:

.. code-block:: python

    BROKER_URL = 'amqp://rabbituser:password@rabbitmqserverhost:5672//'

* Start rabbitmq. (If you installed on
  Linux, it's probably already running, and configured to start automatically
  when Linux restarts.)

After making these changes, everything should be working as before. Verify
that you can schedule tasks and they are executed by the workers.

Monitoring RabbitMQ
-------------------

If you have at least version 3 of RabbitMQ (you might need to install a
newer version than what Debian or Ubuntu provide, see above), then there's
a nice web interface that helps see what is going on with RabbitMQ.

To enable the web interface:

.. code-block:: bash

    $ sudo rabbitmq-plugins enable rabbitmq_management
    The following plugins have been enabled:
      mochiweb
      webmachine
      rabbitmq_mochiweb
      amqp_client
      rabbitmq_management_agent
      rabbitmq_management
    Plugin configuration has changed. Restart RabbitMQ for changes to take effect.
    $ sudo service rabbitmq-server restart
     * Restarting message broker rabbitmq-server                                     [ OK ]
    $

Now you should be able to go to
`http://localhost:15672/`_, log in with the rabbit
user created earlier, and use the management interface.  (If you keep getting
login failures, double-check that the user was tagged `administrator` earlier.)

The `Overview` page is the default when you start the web interface.
Near the top you'll see the count of ready messages, which is the
tasks that are waiting for some worker to execute them:

.. image:: /_static/ready_messages.png

Here there's just one task waiting. If the number of ready messages keeps
going up, you know you might need to start more workers or take other
action.


Workers
-------

In production, you'll still start workers with the command:

.. code-block:: bash

    $ python manage.py celery worker

but you'll probably want to add more options to that command now.

First, by default, celery will start as many worker processes as the server
has processor cores. If that's not enough to keep up, you can run more
processes by adding ``--concurrency=<NUMBER>`` to run NUMBER processes,
and/or run workers on additional servers.

Second, the command by default sends all its output to the screen. Use
``--logfile=</path/to/file>`` to send the output to a file instead.

To see what other options are available:

.. code-block:: bash

    $ python manage.py celery worker --help

Use the techniques described above to control and monitor
the celery workers.

Celerybeat
----------

You'll still want to run just one instance of ``celerybeat``. Use
``--logfile`` to send its output to a file, same as for workers.
To see other options:


.. code-block:: bash

    $ python manage.py celery beat --help

Use the techniques described above to control and monitor
the celerybeat process.


.. _RabbitMQ: http://www.rabbitmq.com
.. _Installing on Debian: http://www.rabbitmq.com/install-debian.html
.. _RabbitMQ download page: http://www.rabbitmq.com/download.html
.. _http://localhost:15672/: http://localhost:15672/
