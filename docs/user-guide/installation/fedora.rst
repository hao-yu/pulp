Fedora 24 - 29
==============

For Fedora 24 through Fedora 29, Pulp is included in the Fedora project.

For Fedora 27 & 28, we recommend to install the upstream repository anyway,
which is usually more up to date.

::

 $ sudo dnf config-manager --add-repo https://repos.fedorapeople.org/repos/pulp/pulp/fedora-pulp.repo

Pulp is no longer included in Fedora 30, or available in an upstream repository for it.
This is because
`MongoDB was removed from Fedora 30. <https://fedoraproject.org/wiki/Changes/MongoDB_Removal>`_

Database Server
---------------

You must provide a running MongoDB instance for Pulp to use. You can use the same host that you
will run Pulp on, or you can give MongoDB its own separate host if you like. You can even use
MongoDB replica sets if you'd like to have higher availability.

::

   $ sudo dnf install mongodb-server

You need mongodb-server with version >= 2.4 installed for Pulp server. MongoDB 2.x reached its EOL.
It is encouraged to use MongoDB 3.x for performance reasons, the recommended version is >= 3.4.
`Installation instructions <https://docs.mongodb.com/v3.6/administration/install-on-linux/>`_
can be found in the MongoDB documentation.

It is highly recommended that you `configure MongoDB to use SSL`_. If you are using
Mongo's authorization feature, you  will need to grant the ``readWrite`` and ``dbAdmin`` roles
to the user you provision for Pulp to use. The ``dbAdmin`` role allows Pulp to create collections
and install indices on them.

After installing MongoDB, you should configure it to start at boot and start it::

 $ sudo systemctl enable mongod
 $ sudo systemctl start mongod

.. warning::
   On new MongoDB installations, MongoDB takes some time to preallocate large files and will not
   accept connections until it finishes. When this happens, Pulp will wait for MongoDB to
   become available before starting.

.. _configure MongoDB to use SSL: http://docs.mongodb.org/v2.4/tutorial/configure-ssl/#configure-mongod-and-mongos-for-ssl


Message Broker
--------------

You must also provide a message broker for Pulp to use. Pulp will work with Qpid or RabbitMQ, but
is tested with Qpid and uses Qpid by default. This can be on the same host that you will
run Pulp on, or elsewhere as you please.


qpidd
^^^^^

To install qpidd, run this command on the host you wish to be the message broker:

::

   $ sudo dnf install qpid-cpp-server qpid-cpp-server-linearstore

Pulp uses the ``ANONYMOUS`` Qpid authentication mechanism by default. To
enable username/password-based ``PLAIN`` broker authentication, you will need
to configure SASL with a username/password, and then configure Pulp to use that
username/password. Refer to the Qpid docs on how to configure username/password
authentication using SASL. Once the broker is configured, configure Pulp according
to the docs on using
:ref:`Pulp with Qpid and username/password authentication <pulp-broker-qpid-with-username-password>`.

The server can be *optionally* configured so that it will connect to the broker using SSL by
following the steps defined in the :ref:`Qpid SSL Configuration Guide <qpid-ssl-configuration>`.
By default, Pulp does not expect to use SSL and will connect to the broker using a plain TCP
connection to localhost.

After installing and configuring Qpid, you should configure it to start at boot and start it:

::

   $ sudo systemctl enable qpidd
   $ sudo systemctl start qpidd


RabbitMQ
^^^^^^^^

On the host you wish to use as your message broker, run this command To install RabbitMQ::

   $ sudo dnf install rabbitmq-server

After installing and configuring RabbitMQ, you should configure it to start at boot and start it::

   $ sudo systemctl enable rabbitmq-server
   $ sudo systemctl start rabbitmq-server


Pulp Server
^^^^^^^^^^^

Now we are ready to install and configure the Pulp server!

#. Install the Pulp server, task workers, and dependencies. You can get the packages you need by
   simply installing the plugins you want to use and they will pull in the needed dependencies. The
   following example installs all currently supported plugins, but feel free to season to taste:

   ::

      $ sudo dnf install pulp-rpm-plugins pulp-docker-plugins pulp-ostree-plugins pulp-puppet-plugins pulp-python-plugins

#. Install the gofer adapter you need for the type of message broker you chose. For qpidd::

      $ sudo dnf install python-gofer-qpid qpid-tools

   For RabbitMQ::

      $ sudo dnf install python-gofer-amqp

#. Edit ``/etc/pulp/server.conf``. Most defaults will work, but these are sections you might
   consider looking at before proceeding. Each section is documented in-line.

   * **email** if you intend to have the server send email (off by default)
   * **database** if your database resides on a different host or port. It is strongly recommended
     that you set ssl and verify_ssl to True.
   * **messaging** if your message broker for communication between Pulp components is on a
     different host or if you want to use SSL. For more information on this section refer to the
     :ref:`Pulp Broker Settings Guide <pulp-broker-settings>`.
   * **tasks** if your message broker for asynchronous tasks is on a different host or if you want
     to use SSL. For more information on this section refer to the
     :ref:`Pulp Broker Settings Guide <pulp-broker-settings>`.
   * **server** if you want to change the server's URL components, hostname, or default credentials

#. Generate RSA key pair and SSL CA certificate::

   $ sudo pulp-gen-key-pair
   $ sudo pulp-gen-ca-certificate

#. Initialize Pulp's database. It is important that the broker is running before initializing
   Pulp's database. It is also important to do this before starting Apache or any Pulp services.
   The database initialization needs to be run as the ``apache`` user, which can be done by
   running:

   ::

      $ sudo -u apache pulp-manage-db

   .. note::
      If Apache or Pulp services are already running, restart them after running the
      ``pulp-manage-db`` command.

   .. warning::
      It is recommended that you configure your web server to refuse SSLv3.0. In Apache, you can do
      this by editing ``/etc/httpd/conf.d/ssl.conf`` and configuring the ``SSLProtocol`` directive
      like this::

         SSLProtocol all -SSLv2 -SSLv3

   .. warning::
      It is recommended that the web server only serve Pulp services.

#. Start Apache httpd and set it to start on boot.::

    $ sudo systemctl enable httpd
    $ sudo systemctl start httpd

#. Pulp has a distributed task system that uses `Celery <http://www.celeryproject.org/>`_.
   Begin by configuring, enabling and starting the Pulp workers. To configure the workers, edit
   ``/etc/default/pulp_workers``. That file has inline comments that explain how to use each
   setting. After you've configured the workers, it's time to enable and start them::

      $ sudo systemctl enable pulp_workers
      $ sudo systemctl start pulp_workers

   .. note::

      The pulp_workers systemd unit does not actually correspond to the workers, but it runs a
      script that dynamically generates units for each worker, based on the configured concurrency
      level. You can check on the status of those generated workers by using the
      ``systemctl status`` command. The workers are named with the template
      ``pulp_worker-<number>``, and they are numbered beginning with 0 and up to
      ``PULP_CONCURRENCY - 1``. For example, you can use ``sudo systemctl status pulp_worker-1`` to
      see how the second worker is doing.

#. There are two more services that need to be running.

   On some Pulp system, configure, start and enable the Celerybeat process. This process performs a
   job similar to a cron daemon for Pulp. Edit ``/etc/default/pulp_celerybeat`` to your liking, and
   then enable and start it. Multiple instances of ``pulp_celerybeat`` may run concurrently, which
   will make the Pulp installation more failure tolerant.

   ::

      $ sudo systemctl enable pulp_celerybeat
      $ sudo systemctl start pulp_celerybeat

   Lastly, a ``pulp_resource_manager`` process must be running in the installation. This process
   acts as a task router, deciding which worker should perform certain types of tasks. As with
   ``pulp_celerybeat``, multiple instances of ``pulp_resource_manager`` may be run concurrently on
   separate hosts to increase fault tolerance, however, only one instance will ever be active at a
   time. Should the active instance become unavailable, another instance will take over after some
   delay.

   Edit ``/etc/default/pulp_resource_manager`` to your liking. Then::

      $ sudo systemctl enable pulp_resource_manager
      $ sudo systemctl start pulp_resource_manager


Admin Client
------------

The Pulp Admin Client is used for administrative commands on the Pulp server,
such as the manipulation of repositories and content. The Pulp Admin Client can
be run on any machine that can access the Pulp server's REST API, including the
server itself. It is not a requirement that the admin client be run on a machine
that is configured as a Pulp consumer.

Pulp admin commands are accessed through the ``pulp-admin`` script.


#. Install the Pulp admin client extentions for the plugin types you wish to use. They depend on
   pulp-admin itself so you will get that along with them. The following example installs all the
   currently available admin extensions, feel free to season to taste:

   ::

      $ sudo dnf install pulp-docker-admin-extensions pulp-puppet-admin-extensions pulp-rpm-admin-extensions pulp-ostree-admin-extensions pulp-python-admin-extensions

#. Update the admin client configuration to point to the Pulp server. Keep in mind
   that because of SSL verification this should be the fully qualified name of the server,
   even if it is the same machine (localhost will not work with the default apache
   generated SSL certificate). Regardless, the "host" setting below must match the
   "CN" value of the server's HTTP SSL certificate.
   This change is made globally to the ``/etc/pulp/admin/admin.conf`` file, or
   for one user in ``~/.pulp/admin.conf``:

   ::

      [server]
      host = localhost.localdomain


Consumer Client and Agent
-------------------------

The Pulp Consumer Client is present on all systems that wish to act as a consumer
of a Pulp server. The Pulp Consumer Client provides the means for a system to
register and configure itself with a Pulp server. Additionally, the Pulp Consumer
Client runs an agent that will receive messages and commands from the Pulp server.

Pulp consumer commands are accessed through the ``pulp-consumer`` script. This
script must be run as root to permit access to add references to the Pulp server's
repositories.

#. Install the Gofer bindings for the message broker you are using. For qpidd::

      $ sudo dnf install python-gofer-qpid

   For RabbitMQ::

      $ sudo dnf install python-gofer-amqp

#. Install the consumer client extensions you wish to use. At the time of this writing, only the RPM
   and Puppet plugins support the Pulp Agent.

   ::

      $ sudo dnf install pulp-puppet-consumer-extensions pulp-rpm-consumer-extensions


#. Update the consumer client configuration to point to the Pulp server. Keep in mind
   that because of the SSL verification this should be the fully qualified name of the server,
   even if it is the same machine (localhost will not work with the default Apache
   generated SSL certificate). Regardless, the "host" setting below must match the
   "CN" value of the server's HTTP SSL certificate.
   This change is made to the ``/etc/pulp/consumer/consumer.conf`` file:

   ::

      [server]
      host = localhost.localdomain

#. The agent may be configured so that it will connect to the Qpid broker using SSL by
   following the steps defined in the :ref:`Qpid SSL Configuration Guide <qpid-ssl-configuration>`.
   By default, the agent will connect using a plain TCP connection.

#. Set the agent to start at boot::

      $ sudo systemctl enable goferd
      $ sudo systemctl start goferd


Extra Configuration
-------------------

You are now ready to proceed to :doc:`extra_configuration`.
