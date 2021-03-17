=========
Hitch Fix
=========

When installing varnish and hitch manually to a cPanel server, hitch needs to
know about each SSL file on the server. Typically when new SSLs are added or
removed the corresponding SSL path needs to be added or removed from the hitch
configuration. This guide explains how to automate adding and removing SSL
paths from the hitch configuration.

Procedure
---------

This assumes the hitch configuration file is located
``/etc/hitch/hitch.conf``. Be sure to verify the location of the hitch.conf
file.

#.  Backup current ``hitch.conf``::

     cp -v /etc/hitch/hitch.conf{,.bk$(date +%F)}

#.  Install inotify-tools::

     yum -y install inotify-tools

#.  Create ``/usr/local/src/hitchfix.sh`` with::

     #!/usr/bin/bash
     sed -i '/pem-file/d' /etc/hitch/hitch.conf
     for ssl in /var/cpanel/ssl/apache_tls/*/combined
         do echo "pem-file = \"${ssl}\"" >> /etc/hitch/hitch.conf
     done
     systemctl restart hitch

#.  Make ``/usr/local/src/hitchfix.sh`` executable::

     chmod +x /usr/local/src/hitchfix.sh

#.  Once the above is complete, run the script to be sure it updates
    ``hitch.conf`` as expected::

        sh /usr/local/src/hitchfix.sh

#.  Once verified the script operates, run it using ``inotifywait`` once to start
    the process::

        /usr/bin/inotifywait /var/cpanel/ssl/apache_tls -qrm --event attrib | \
        while read change; do /bin/sh /usr/local/src/hitchfix.sh; done &

#.  Finally, create a root cron job to keep the job alive on server reboot::

        @reboot /usr/bin/inotifywait /var/cpanel/ssl/apache_tls -qrm --event attrib | while read change; do /bin/sh /usr/local/src/hitchfix.sh; done &

