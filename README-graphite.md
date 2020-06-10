This document describes how to set up a Graphite and Grafana network monitoring solution for a Debian based system.

Based on these instructions:

* [Graphite project](http://graphite.readthedocs.io/en/latest/index.html)
* [*Monitoring with Graphite*](http://shop.oreilly.com/product/0636920035794.do) book
* Debian package documentation for graphite-web, in `/usr/share/doc/graphite-web/README.debian`
* [Grafana docs](http://shop.oreilly.com/product/0636920035794.do)


Graphite
========

For a Debian-based system, install these packages:

* build-essential
* python3-dev
* graphite-carbon (installs python3-whisper)
* graphite-web (installs python3-django); must use at least v1.1.4-3
* apache2 (used by graphite-web)
* libapache2-mod-wsgi-py3

Preliminaries
-------------
I installed on an Ubuntu 2020.04 workstation, which includes Python 3.8. Graphite v1.1.6 was updated for this version of Python. Unfortunately the Debian graphite components still are at v1.1.4, so we must adjust the installation.

The issue is with the graphite-web package. We must update update a few files, as described in [PR #2464](https://github.com/graphite-project/graphite-web/pull/2464/files) for the graphite-project/graphite-web GitHub repository.

Update the files below (relative to `/usr/lib/python3/dist-packages/graphite`) to the content in that PR:

* `render/functions.py`
* `render/views.py`
* `util.py`


Configuration
-------------

**`/etc/carbon/storage-schemas.conf`**

Configure interval and retention. Below is a good starting point. Entries are scanned in order in this file, so must insert *before* a matching entry with a generic pattern!

```
[temp]
pattern = ^temp\.
retentions = 60s:90d
```

**`/etc/graphite/local_settings.py`**

Update the following settings for your local setup.

```
SECRET_KEY = 'somelonganduniquesecretstring'
TIME_ZONE = 'America/New_York'
```

Also, the database setup routine in the next section required that the Django INSTALLED_APPS setting was populated. This setting is not initiailized in the `global_settings.py` for the python3-django package, so we initialize it in `local_settings.py`.

Note: python3-django uses v2.2 of Django. It may be that the graphite-web package has not caught up fully to this version of Django, and so we must manually initialize this stting.
```
# Uncomment the following line for direct access to Django settings such as
# MIDDLEWARE or APPS
#from graphite.app_settings import *
from graphite.app_settings import *

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'graphite.account',
    'graphite.dashboard',
    'graphite.events',
    'graphite.tags',
    'graphite.url_shortener',
    'tagging'
]
```

**Django database**

Initialize the Django graphite database:

```
$ sudo su -s /bin/bash _graphite -c 'graphite-manage migrate'
```

When I ran the above command, the output was as shown below.
```
Operations to perform:
  Apply all migrations: account, admin, auth, contenttypes, dashboard, events, sessions, tagging, tags, url_shortener
Running migrations:
  Applying contenttypes.0001_initial... OK
  Applying auth.0001_initial... OK
  Applying account.0001_initial... OK
  Applying admin.0001_initial... OK
  Applying admin.0002_logentry_remove_auto_add... OK
  Applying admin.0003_logentry_add_action_flag_choices... OK
  Applying contenttypes.0002_remove_content_type_name... OK
  Applying auth.0002_alter_permission_name_max_length... OK
  Applying auth.0003_alter_user_email_max_length... OK
  Applying auth.0004_alter_user_username_opts... OK
  Applying auth.0005_alter_user_last_login_null... OK
  Applying auth.0006_require_contenttypes_0002... OK
  Applying auth.0007_alter_validators_add_error_messages... OK
  Applying auth.0008_alter_user_username_max_length... OK
  Applying auth.0009_alter_user_last_name_max_length... OK
  Applying auth.0010_alter_group_name_max_length... OK
  Applying auth.0011_update_proxy_permissions... OK
  Applying dashboard.0001_initial... OK
  Applying events.0001_initial... OK
  Applying sessions.0001_initial... OK
  Applying tagging.0001_initial... OK
  Applying tagging.0002_on_delete... OK
  Applying tags.0001_initial... OK
  Applying url_shortener.0001_initial... OK
```

**Apache**

Set up the Graphite web UI in Apache although really we only need the web UI as a web service.
```
$ sudo cp /usr/share/graphite-web/apache2-graphite.conf /etc/apache2/sites-available

```

Finally, disable the default Apache site and enable the Graphite site:
```
$ sudo a2dissite 000-default
$ sudo a2ensite apache2-graphite
$ sudo systemctl reload apache2
```

Running
-------

### Ports

* 2003 TCP/UDP -- Plaintext input
* 2004 TCP -- Pickle input
* 7002 -- Cache query for output

### Data

* `/var/lib/graphite/whisper`

The tree of stored metrics is below the `whisper` directory. You can review the entries to verify the expected data has been saved.

### Logs

* `/var/log/carbon`
* `/var/log/apache2/graphite-web_access.log`
* `/var/log/apache2/graphite-web_error.log`
* `/var/log/graphite`
`

In `/var/log/carbon/console.log`, you may see lines like below. Since `storage-aggregation.conf` is optional, the lines are not an issue.

```
01/01/2020 12:59:37 :: /etc/carbon/storage-aggregation.conf not found or wrong perms, ignoring.
```

Users:

* _graphite

Metric Names
------------

Graphite uses a dotted name for each metric, like `temp.3303.0`. This name refers to the temperature reading for instance 0. 

It is important to structure the metric name in a consistent hierarchical fashion. The first part of the name should contain static elements, and variable elements like node names should appear later. Matt Aimonetti's [post](https://matt.aimonetti.net/posts/2013/06/26/practical-guide-to-graphite-monitoring/)
provides useful guidelines.


Grafana Dashboard
=================

Install from the [download instructions](https://grafana.com/grafana/download). See the [installation instructions](http://docs.grafana.org/installation/debian/) for file locations.

Configuration
-------------

Create a data source for temp in the grafana dashboard. Be sure to use 'proxy' access to the data source.

Running
-------

The commands below, for systemd, start Grafana and setup to start on boot.

```
$ sudo systemctl daemon-reload
$ sudo systemctl enable grafana-server
$ sudo systemctl start grafana-server
$ sudo systemctl status grafana-server
```

Access at `http://localhost:3000`

View the log: `/var/log/grafana/grafana.log`
