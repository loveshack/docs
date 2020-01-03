.. _onegate_configure:

=============================
OneGate Server Configuration
=============================

The OneGate service allows Virtual Machine guests to pull and push VM information from OpenNebula. Although it is installed by default, its use is completely optional.

Requirements
============

Check the :ref:`Installation guide <ignc>` for details of what package you have to install depending, on your distribution.

OneGate can be used with VMs running on :ref:`KVM, vCenter <context_overview>`, and :ref:`EC2 <context_ec2>`.

Currently, OneGate is not supported for VMs instantiated in Azure, since the authentication token is not available inside these VMs. OneGate support for these drivers will be included in upcoming releases.

Configuration
=============

The OneGate configuration file can be found at ``/etc/one/onegate-server.conf``. It uses YAML syntax to define the following options:

**Server Configuration**

* ``one_xmlrpc``: OpenNebula daemon host and port
* ``host``: Host where OneGate will listen
* ``port``: Port where OneGate will listen
* ``ssl_server``: SSL proxy URL that serves the API (set if it is being used)

**Log**

* ``debug_level``: Log debug level. 0 = ERROR, 1 = WARNING, 2 = INFO, 3 = DEBUG

**Auth**

* ``auth``: Authentication driver for incoming requests.

  * ``onegate``: Based on token provided in the context

* ``core_auth``: Authentication driver to communicate with OpenNebula core.

  * ``cipher`` for symmetric cipher encryption of tokens
  * ``x509`` for x509 certificate encryption of tokens. For more information, visit the :ref:`OpenNebula Cloud Auth documentation <cloud_auth>`.

* ``oneflow_server`` Endpoint where the OneFlow server is listening.
* ``permissions`` By default OneGate exposes all the available API calls. Each of the actions can be enabled/disabled in the server configuration.
* ``restricted_attrs`` Attributes that cannot be modified when updating a VM template
* ``restricted_actions`` Actions that cannot be performed on a VM

This is the default file

.. code-block:: yaml

    ################################################################################
    # Server Configuration
    ################################################################################

    # OpenNebula sever contact information
    #
    :one_xmlrpc: http://localhost:2633/RPC2

    # Server Configuration
    #
    :host: 127.0.0.1
    :port: 5030

    # SSL proxy URL that serves the API (set if is being used)
    #:ssl_server: https://service.endpoint.fqdn:port/

    ################################################################################
    # Log
    ################################################################################

    # Log debug level
    #   0 = ERROR, 1 = WARNING, 2 = INFO, 3 = DEBUG
    #
    :debug_level: 3

    ################################################################################
    # Auth
    ################################################################################

    # Authentication driver for incomming requests
    #   onegate, based on token provided in the context
    #
    :auth: onegate

    # Authentication driver to communicate with OpenNebula core
    #   cipher, for symmetric cipher encryption of tokens
    #   x509, for x509 certificate encryption of tokens
    #
    :core_auth: cipher

    ################################################################################
    # OneFlow Endpoint
    ################################################################################

    :oneflow_server: http://localhost:2474

    ################################################################################
    # Permissions
    ################################################################################

    :permissions:
      :vm:
        :show: true
        :show_by_id: true
        :update: true
        :update_by_id: true
        :action_by_id: true
      :service:
        :show: true
        :change_cardinality: true

    # Attrs that cannot be modified when updating a VM template
    :restricted_attrs
      - SCHED_REQUIREMENTS
      - SERVICE_ID
      - ROLE_NAME

    # Actions that cannot be performed on a VM
    :restricted_actions
      #- deploy
      #- delete
      #- hold
      ...

Start OneGate
=============

To start and stop the server, use the ``opennebula-gate`` service:

.. prompt:: bash # auto

    # systemctl start opennebula-gate

Or use ``service`` in older Linux systems:

.. prompt:: bash # auto

    # service opennebula-gate start

.. warning:: By default, the server will only listen to requests coming from localhost. Change the ``:host`` attribute in ``/etc/one/onegate-server.conf`` to your server public IP, or 0.0.0.0 so OneGate will listen on any interface.

Inside ``/var/log/one/`` you will find new log files for the server:

.. code::

    /var/log/one/onegate.error
    /var/log/one/onegate.log

Use OneGate
===========

Before your VMs can communicate with OneGate, you need to edit ``/etc/one/oned.conf`` and set the OneGate endpoint. This IP must be reachable from your VMs.

.. code::

    ONEGATE_ENDPOINT = "http://192.168.0.5:5030"

At this point the service is ready and you can continue to the :ref:`OneGate usage documentation <onegate_usage>`.

Configuring an SSL Proxy
========================

You can configure Nginx or lighttpd as an SSL proxy for OneGate similarly to the :ref:`setup for Sunstone <ss_proxy>`.

After installing Nginx and a certificate, edit the default Nginx configuration file or generate a new one similarly to this example. Change the ONEGATE_ENDPOINT variable to your own domain name.

.. code::

    server {
      listen 80;
      return 301 https://$host$request_uri;
    }

    server {
      listen 443;
      server_name ONEGATE_ENDPOINT;

      ssl_certificate           /etc/one/cert.crt;
      ssl_certificate_key       /etc/one/cert.key;

      ssl on;
      ssl_session_cache  builtin:1000  shared:SSL:10m;
      ssl_protocols  TLSv1 TLSv1.1 TLSv1.2;
      ssl_ciphers HIGH:!aNULL:!eNULL:!EXPORT:!CAMELLIA:!DES:!MD5:!PSK:!RC4;
      ssl_prefer_server_ciphers on;

      access_log            /var/log/nginx/onegate.access.log;

      location / {

        proxy_set_header        Host $host;
        proxy_set_header        X-Real-IP $remote_addr;
        proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header        X-Forwarded-Proto $scheme;

        # Fix the “It appears that your reverse proxy set up is broken" error.
        proxy_pass          http://localhost:5030;
        proxy_read_timeout  90;

        proxy_redirect      http://localhost:5030 https://ONEGATE_ENDPOINT;
      }
    }

Update ``/etc/one/oned.conf`` with the new OneGate endpoint.

.. code::

    ONEGATE_ENDPOINT = "https://ONEGATE_ENDPOINT"


Update ``/etc/one/onegate-server.conf`` with the new OneGate endpoint and uncomment the ``ssl_server`` parameter:

.. code::

    :ssl_server: https://ONEGATE_ENDPOINT

Then restart oned, onegate-server and Nginx:

.. prompt:: bash $ auto

    $ sudo service nginx restart
    $ sudo service opennebula restart
    $ sudo service opennebula-gate restart
