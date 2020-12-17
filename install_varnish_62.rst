=====================================================
Installing and configuring Varnish, Hitch, and Apache
=====================================================

This guide covers how to install Varnish and Hitch into a CentOS 7 server and
assumes the server is currently running just Apache.

Procedure
---------

Install Varnish
~~~~~~~~~~~~~~~

Varnish repos for CentOS are located https://packagecloud.io/varnishcache/

To install the RPM, run (pay attention to the version needed)::

    curl -s https://packagecloud.io/install/repositories/varnishcache/varnish62/script.rpm.sh | sudo bash

You should be able to find Varnish now::

    $ yum search varnish
    collectd-varnish.x86_64 : Varnish plugin for collectd
    varnish-devel.x86_64 : Development files for varnish
    varnish-docs.x86_64 : Documentation files for varnish
    varnish-libs.x86_64 : Libraries for varnish
    varnish-libs-devel.x86_64 : Development files for varnish-libs
    varnish.x86_64 : High-performance HTTP accelerator

Install Varnish:
``yum install varnish``

Install Hitch
~~~~~~~~~~~~~

To handle SSL connections, an SSL terminator is needed because Varnish only handles HTTP. Hitch is an example of one that will work.

Hitch is located in CentOS's EPEL repository.

Install EPEL:
``yum install epel-release``

Install Hitch:
``yum install hitch``

Hitch's configuration file is located ``/etc/hitch/hitch.conf`` and the default
configuration is sufficient, except that it will need at least one ``pem-file``
definition with a path to an SSL file.

NOTE! - Hitch will not start if no SSL file or folder path has been specified!

Obtain the certificate files from Apache by using:
``grep SSLCertificateFile /etc/apache2/conf/httpd.conf``

Add ``pem-file`` lines for each SSL certificate to ``/etc/hitch/hitch.conf``.

Create Varnish systemctl unit file
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Varnish's systemctl file will need to be updated.

Here is a good example of one::

    [Unit]
    Description=Varnish Cache, a high-performance HTTP accelerator
    After=network-online.target
    
    [Service]
    Type=forking
    KillMode=process
    
    # Maximum number of open files (for ulimit -n)
    LimitNOFILE=131072
    
    # Locked shared memory - should suffice to lock the shared memory log
    # (varnishd -l argument)
    # Default log size is 80MB vsl + 1M vsm + header -> 82MB
    # unit is bytes
    LimitMEMLOCK=85983232
    
    # Enable this to avoid "fork failed" on reload.
    TasksMax=infinity
    
    # Maximum size of the corefile.
    LimitCORE=infinity
    
    ExecStart=/usr/sbin/varnishd \
      -a :80 \
      -a 127.0.0.1:6086,PROXY \
      -f /etc/varnish/m2.vcl \
      -s malloc,256m \
    ExecReload=/usr/sbin/varnishreload
    
    [Install]
    WantedBy=multi-user.target

Note the ExecStart lines. Varnish will listen on port 80 in this example as
well as port 6086. The purpose of listening on ``127.0.0.1:6086`` is to
receive proxy connections from Hitch. The default Varnish VCL file was in
this case changed from ``/etc/varnish/default.vcl`` to ``/etc/varnish/m2.vcl``
for Magento 2.

Varnish Considerations
~~~~~~~~~~~~~~~~~~~~~~

Varnish will not cache out of the box. It will need to be configured to do so.

Varnish allows great depth of customization through VCL. The main Varnish VCL
file is not sufficient to start caching requests and will need to be modified.
It is suggested to get started with the `Varnish VCL
<https://varnish-cache.org/docs/6.0/users-guide/vcl.html>`_ guide.

Multiple IPs in Apache
----------------------

If Apache has virtual hosts across several IPs, then Varnish will need to also
be configured appropriately.

In the main varnish VCL, multiple backends should be set::

    backend server1 {
        .host = "IP-ADDRESS-1";
        .port = "8080";
    }
    backend server2 {
        .host = "IP-ADDRESS-2";
        .port = "8080";
    }

And in the vcl_recv function, Varnish needs to know what backend to set for the requested domain::

    sub vcl_recv {
        if (server.ip == "IP-ADDRESS-1") {
            set req.backend = server1; 
        } else {
            set req.backend = server2;
        }
        ...
    }

Varnish and Magento
-------------------

Magento 2 allows the ability to generate a Varnish VCL file for you and I suggest this route. This can be done from CLI or the Magento 2 backend. Once that is generated, the ExecStart systemctl line will need to be updated to reflect the new VCL file.

Allow Magento to purge Varnish cache
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When Magento is behind Varnish, it is possible to allow a Magento site to
purge the Varnish cache.

There are a few configuration changes that are needed.

In the Varnish VCL, the purge acl should contain the IP of the website, so the
website can send purge requests to itself::

    acl purge {
        "localhost";
        "50.50.50.50";
    }

This says localhost and are allowed to send purge requests.

Magento also needs to be configured with an 'http_cache_host'.

From https://devdocs.magento.com/guides/v2.4/config-guide/varnish/use-varnish-cache.html::

    Magento purges Varnish hosts after you configure Varnish hosts using the magento setup:config:set command.

    You can use the optional parameter --http-cache-hosts parameter to specify a comma-separated list of Varnish hosts and listen ports. Configure all Varnish hosts, whether you have one or many. (Do not separate     hosts with a space character.)

    The parameter format must be <hostname or ip>:<listen port>, where you can omit <listen port> if it’s port 80.

The command to set the cache hosts looks something like::

    bin/magento setup:config:set --http-cache-hosts=192.0.2.100,192.0.2.155:6081


Infinite https redirect loop with Magento 2 and Varnish
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

I encountered this issue and the reason it occured is because Varnish looks for an HTTP header called X-Forwarded-Proto to determine if the request has come in over http or https. The header needs to be set for Magento to know on what protocol the request came in.

This was addressed by adding the following to the VCL file ( in the sub vcl_recv { block )::

    # if request comes from hitch, set X-Forwarded-Proto header to https
    if (std.port(local.ip) == 6086) {
        set req.http.X-Forwarded-Proto = "https";

For the above to work, there must also be this line in the VCL file ( this is not included in the default VCL file)::

    import std;


-------------------------------------

here's an example magento 2 VCL file::

    vcl 4.0;
    
    import std;
    # The minimal Varnish version is 4.0
    # For SSL offloading, pass the following header in your proxy server or load balancer: 'X-Forwarded-Proto: https'
    
    backend default {
        .host = "50.50.50.50";
        .port = "8080";
    # TODO: look into this
    # varnish errors out unless this is commented out
    # I am not sure why this is the case!
    #    .first_byte_timeout = 600s;
    #    .probe = {
    #        #.url = "/pub/health_check.php";
    #        .url = "/health_check.php";
    #        .timeout = 2s;
    #        .interval = 5s;
    #        .window = 10;
    #        .threshold = 5;
    #   }
    }
    
    # if multiple IPs are present, multiple backends are needed
    # in vcl_recv will need to specify the backend based on the
    # requested website
    
    
    acl purge {
        "localhost";
    }
    sub vcl_recv {
    
        # if request comes from hitch, set X-Forwarded-Proto header to https
        if (std.port(local.ip) == 6086) {
            set req.http.X-Forwarded-Proto = "https";
        }
    
        if (req.method == "PURGE") {
            if (client.ip !~ purge) {
                return (synth(405, "Method not allowed"));
            }
            # To use the X-Pool header for purging varnish during automated deployments, make sure the X-Pool header
            # has been added to the response in your backend server config. This is used, for example, by the
            # capistrano-magento2 gem for purging old content from varnish during it's deploy routine.
            if (!req.http.X-Magento-Tags-Pattern && !req.http.X-Pool) {
                return (synth(400, "X-Magento-Tags-Pattern or X-Pool header required"));
            }
            if (req.http.X-Magento-Tags-Pattern) {
              ban("obj.http.X-Magento-Tags ~ " + req.http.X-Magento-Tags-Pattern);
            }
            if (req.http.X-Pool) {
              ban("obj.http.X-Pool ~ " + req.http.X-Pool);
            }
            return (synth(200, "Purged"));
        }
    
        if (req.method != "GET" &&
            req.method != "HEAD" &&
            req.method != "PUT" &&
            req.method != "POST" &&
            req.method != "TRACE" &&
            req.method != "OPTIONS" &&
            req.method != "DELETE") {
              /* Non-RFC2616 or CONNECT which is weird. */
              return (pipe);
        }
    
        # We only deal with GET and HEAD by default
        if (req.method != "GET" && req.method != "HEAD") {
            return (pass);
        }
    
        # Bypass shopping cart, checkout and search requests
        #if (req.url ~ "/checkout" || req.url ~ "/catalogsearch") {
            if (req.url ~ "/checkout") {
            return (pass);
        }
    
        # Bypass health check requests
        if (req.url ~ "/pub/health_check.php") {
            return (pass);
        }
    
        # Set initial grace period usage status
        set req.http.grace = "none";
    
        # normalize url in case of leading HTTP scheme and domain
        set req.url = regsub(req.url, "^http[s]?://", "");
    
        # collect all cookies
        std.collect(req.http.Cookie);
    
        # Compression filter. See https://www.varnish-cache.org/trac/wiki/FAQ/Compression
        if (req.http.Accept-Encoding) {
            if (req.url ~ "\.(jpg|jpeg|png|gif|gz|tgz|bz2|tbz|mp3|ogg|swf|flv)$") {
                # No point in compressing these
                unset req.http.Accept-Encoding;
            } elsif (req.http.Accept-Encoding ~ "gzip") {
                set req.http.Accept-Encoding = "gzip";
            } elsif (req.http.Accept-Encoding ~ "deflate" && req.http.user-agent !~ "MSIE") {
                set req.http.Accept-Encoding = "deflate";
            } else {
                # unknown algorithm
                unset req.http.Accept-Encoding;
            }
        }
    
        # Remove all marketing get parameters to minimize the cache objects
        if (req.url ~ "(\?|&)(gclid|cx|ie|cof|siteurl|zanpid|origin|fbclid|mc_[a-z]+|utm_[a-z]+|_bta_[a-z]+)=") {
            set req.url = regsuball(req.url, "(gclid|cx|ie|cof|siteurl|zanpid|origin|fbclid|mc_[a-z]+|utm_[a-z]+|_bta_[a-z]+)=[-_A-z0-9+()%.]+&?", "");
            set req.url = regsub(req.url, "[?|&]+$", "");
        }
    
        # Static files caching
        if (req.url ~ "^/(pub/)?(media|static)/") {
            # Static files should not be cached by default
            #return (pass);
    
            # But if you use a few locales and don't use CDN you can enable caching static files by commenting previous line (#return (pass);) and uncommenting next 3 lines
            unset req.http.Https;
            unset req.http.X-Forwarded-Proto;
            unset req.http.Cookie;
        }
    
        return (hash);
    }
    
    sub vcl_hash {
        if (req.http.cookie ~ "X-Magento-Vary=") {
            hash_data(regsub(req.http.cookie, "^.*?X-Magento-Vary=([^;]+);*.*$", "\1"));
        }
    
        # For multi site configurations to not cache each other's content
        if (req.http.host) {
            hash_data(req.http.host);
        } else {
            hash_data(server.ip);
        }
    
        if (req.url ~ "/graphql") {
            call process_graphql_headers;
        }
    
        # To make sure http users don't see ssl warning
        if (req.http.X-Forwarded-Proto) {
            hash_data(req.http.X-Forwarded-Proto);
        }
    
    }
    
    sub process_graphql_headers {
        if (req.http.Store) {
            hash_data(req.http.Store);
        }
        if (req.http.Content-Currency) {
            hash_data(req.http.Content-Currency);
        }
    }
    
    sub vcl_backend_response {
    
        set beresp.grace = 3d;
    
        if (beresp.http.content-type ~ "text") {
            set beresp.do_esi = true;
        }
    
        if (bereq.url ~ "\.js$" || beresp.http.content-type ~ "text") {
            set beresp.do_gzip = true;
        }
    
    if (bereq.url ~ "/catalogsearch") {
        set beresp.ttl = 30m;
    }
        #if (beresp.http.X-Magento-Debug) {
        #    set beresp.http.X-Magento-Cache-Control = beresp.http.Cache-Control;
        ##}
    
        # cache only successfully responses and 404s
        if (beresp.status != 200 && beresp.status != 404) {
            set beresp.ttl = 0s;
            set beresp.uncacheable = true;
            return (deliver);
        } elsif (beresp.http.Cache-Control ~ "private") {
            set beresp.uncacheable = true;
            set beresp.ttl = 86400s;
            return (deliver);
        }
    
        # validate if we need to cache it and prevent from setting cookie
        if (beresp.ttl > 0s && (bereq.method == "GET" || bereq.method == "HEAD")) {
            unset beresp.http.set-cookie;
        }
       # If page is not cacheable then bypass varnish for 2 minutes as Hit-For-Pass
       if (beresp.ttl <= 0s ||
           beresp.http.Surrogate-control ~ "no-store" ||
           (!beresp.http.Surrogate-Control &&
           beresp.http.Cache-Control ~ "no-cache|no-store") ||
           beresp.http.Vary == "*") {
           # Mark as Hit-For-Pass for the next 2 minutes
            set beresp.ttl = 120s;
            set beresp.uncacheable = true;
        }
    
        return (deliver);
    }
    
    sub vcl_deliver {
        if (resp.http.X-Magento-Debug) {
            if (resp.http.x-varnish ~ " ") {
                set resp.http.X-Magento-Cache-Debug = "HIT";
                set resp.http.Grace = req.http.grace;
            } else {
                set resp.http.X-Magento-Cache-Debug = "MISS";
            }
        } else {
            #unset resp.http.Age;
        }
    
        # Not letting browser to cache non-static files.
       if (resp.http.Cache-Control !~ "private" && req.url !~ "^/(pub/)?(media|static)/") {
            set resp.http.Pragma = "no-cache";
            set resp.http.Expires = "-1";
            set resp.http.Cache-Control = "no-store, no-cache, must-revalidate, max-age=0";
        }
    
        unset resp.http.X-Magento-Debug;
        unset resp.http.X-Magento-Tags;
        unset resp.http.X-Powered-By;
        unset resp.http.Server;
        unset resp.http.X-Varnish;
        unset resp.http.Via;
        unset resp.http.Link;
    }
    
    sub vcl_hit {
        if (obj.ttl >= 0s) {
            # Hit within TTL period
            return (deliver);
        }
        if (std.healthy(req.backend_hint)) {
            if (obj.ttl + 300s > 0s) {
                # Hit after TTL expiration, but within grace period
                set req.http.grace = "normal (healthy server)";
                return (deliver);
            } else {
                # Hit after TTL and grace expiration
                return (pass);
            }
        } else {
            # server is not healthy, retrieve from cache
            set req.http.grace = "unlimited (unhealthy server)";
            return (deliver);
        }
    }
