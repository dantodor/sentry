Throttles and Rate Limiting
===========================

With the way Sentry works you may find yourself in a situation where
you'll see too much inbound traffic without a good way to drop excess
messages. There's a few solutions to this, and you'll likely want to
employ them all if you are faced with this problem.

Event Quotas
------------

One of the primary mechanisms for throttling workloads in Sentry involves
setting up event quotas. These can be configured per project as well as
system wide and will allow you to limit the maximum number of events
accepted within a 60 second period of time.

Configuration
`````````````

The primary implementation uses Redis, and simply requires you to configure
the connection information:

.. code-block:: python

    SENTRY_QUOTAS = 'sentry.quotas.redis.RedisQuota'
    SENTRY_QUOTA_OPTIONS = {
        'hosts': {
            0: {
                'host': 'localhost',
                'port': 6379
            }
        }
    }

You also have the ability to specify multiple nodes and have keys automatically
distributed. It's unlikely that you'll need this functionality, but if you do, a simple
configuration might look like this:

.. code-block:: python

    SENTRY_QUOTA_OPTIONS = {
        'hosts': {
            0: {
                'host': '192.168.1.1'
            },
            1: {
                'host': '192.168.1.2'
            }
        },
    }


You can also configure system-wide maximums, and a default value for all projects:

.. code-block:: python

   SENTRY_DEFAULT_MAX_EVENTS_PER_MINUTE = '90%'
   SENTRY_SYSTEM_MAX_EVENTS_PER_MINUTE = 500

If you have additional needs, you're freely available to extend the base
Quota class just as the Redis implementation does.

Notification Rate Limits
------------------------

In some cases there may be concerns about limiting things such as outbound email
notifications. To address this Sentry provides a rate limits subsystem which supports
arbitrary rate limits.

Configuration
`````````````

Like event quotas, the primary implementation uses Redis:


.. code-block:: python

    SENTRY_RATELIMITER = 'sentry.ratelimits.redis.RedisRateLimiter'
    SENTRY_RATELIMITER_OPTIONS = {
        'hosts': {
            0: {
                'host': 'localhost',
                'port': 6379
            }
        }
    }


Rate Limiting with IPTables
---------------------------

One of your most effective options is to rate limit with your system's
firewall, in our case, IPTables. If you're not sure how IPTables works,
take a look at `Ubuntu's IPTables How-to
<https://help.ubuntu.com/community/IptablesHowTo>`_.

A sampe configuration, which will limit a single IP from bursting more
than 5 messages in a 10 second period might look like this::

    # create a new chain for rate limiting
    -N LIMITED

    # rate limit individual ips to prevent stupidity
    -I INPUT -p tcp --dport 80 -m state --state NEW -m recent --set
    -I INPUT -p tcp --dport 443 -m state --state NEW -m recent --set
    -I INPUT -p tcp --dport 80 -m state --state NEW -m recent --update --seconds 10 --hitcount 5 -j LIMITED
    -I INPUT -p tcp --dport 443 -m state --state NEW -m recent --update --seconds 10 --hitcount 5 -j LIMITED

    # log rejected ips
    -A LIMITED -p tcp -m limit --limit 5/min -j LOG --log-prefix "Rejected TCP: " --log-level 7
    -A LIMITED -j REJECT

Rate Limiting with Nginx
------------------------

While IPTables will help prevent DDOS they don't effectively communicate
to the client that it's being rate limited. This can be important
depending on how the client chooses to respond to the situation.

An alternative (or rather, an addition) is to use something like
`ngx_http_limit_conn_module
<http://nginx.org/en/docs/http/ngx_http_limit_conn_module.html>`_.

An example configuration looks something like this::

    limit_req_zone  $binary_remote_addr  zone=one:100m   rate=3r/s;
    limit_req_zone  $projectid  zone=two:100m   rate=6r/s;
    limit_req_status 429;
    limit_req_log_level warn;

    server {
      listen   80;

      location / {
        proxy_pass        http://internal;
      }

      location ~* /api/(?P<projectid>\d+/)?store/ {
        proxy_pass        http://internal;

        limit_req   zone=one  burst=3  nodelay;
        limit_req   zone=two  burst=10  nodelay;
      }
    }

Using Cyclops (Client Proxy)
----------------------------

An additional option for rate limiting is to do it on the client side.
`Cyclops <https://github.com/heynemann/cyclops>`_ is a third-party proxy
written in Python (using Tornado) which aims to solve this.

It's not officially supported, however it is used in production by several
large users.