weave-minimal: a Firefox Sync Server that just works™
=====================================================

**DOES NOT WORK WITH FIREFOX 29 OR LATER**

This is a lightweight implementation of Mozillas' [User API v1.0][1] and
[Storage API v1.1][2] without LDAP, MySQL, Redis etc. overhead. It is multi
users capable and depends only on [werkzeug][3].

I mean, *really* lightweight and *really* simple to install. No hg-attack clone
fetch fail apt-get install. It just works.

Note, that the name originates from the deprecated [Weave Minimal Server][4],
but shares nothing beside the name; see [FSyncMS][5] for a still working, PHP
based sync server.

[1]: http://docs.services.mozilla.com/reg/apis.html
[2]: http://docs.services.mozilla.com/storage/apis-1.1.html
[3]: http://werkzeug.pocoo.org/
[4]: https://tobyelliott.wordpress.com/2011/03/25/updating-and-deprecating-the-weave-minimal-server/
[5]: https://github.com/balu-/FSyncMS/

Setup and Configuration
-----------------------

You need Python 2.6, 2.7 or 3.3. See `weave-minimal --help` for a list of
parameters and a short description.

    $ pip install weave-minimal
    $ weave-minimal --enable-registration
     * Running on http://127.0.0.1:8080/

To run weave-minimal as a daemon, consider this SysVinit script:

```bash
$ cat /etc/init.d/weave-minimal
#!/bin/sh

NAME=weave-minimal
USER=www
CMD=/usr/local/bin/weave-minimal

PORT=8080
DBPATH=/var/lib/weave-minimal/

if [ ! -d $DBPATH ]; then
  mkdir /var/lib/weave-minimal
  chown $USER /var/lib/weave-minimal
fi

case $1 in
    start)
        echo -n "Starting $NAME."
        start-stop-daemon --start --background --pidfile /var/run/$NAME.pid \
        --chuid $USER --make-pidfile --exec $CMD -- --data-dir=$DBPATH \
        --port=$PORT --enable-registration
       ;;
    stop)  start-stop-daemon --stop --pidfile /var/run/$NAME.pid
       ;;
esac
$ chmod +x /etc/init.d/weave-minimal
$ sudo update-rc.d weave-minimal defaults 99
$ sudo invoke-rc.d weave-minimal start
```

### See also

* [EN: Firefox Sync server right on router][6]
* [DE: Uberspace und dein Firefox Sync Server][7], [FastCGI wrapper][8]

[6]: http://forums.smallnetbuilder.com/showthread.php?t=10797
[7]: http://christoph-polcin.com/2012/12/31/firefox-minimal-weave-auf-uberspace/
[8]: https://github.com/oa/weave-minimal-uberspace

Setting up Firefox
------------------

0. **Migrate from the official servers**: write down your email address and sync
   key (you can reset your password anyway) and unlink your client. If you want
   to keep your previous sync key, enter the key in the advanced settings.

1. **Create a new account** in the sync preferences. Choose a valid email
   address and password and enter the custom url into the server location
   (leave the trailing slash!). If you get an error, check the SSL certificate
   first.

2. If no errors come up, click continue and wait a minute. If you sync tabs,
   quit, re-open and manually sync otherwise you'll get an empty tab list.

3. **Connect other clients** is as easy as with the mozilla servers (the client
   actually uses mozilla's servers for this): click *I already have an account*
   and write the three codes into an already linked browser using *Pair Device*.
   Optionally you can use the manual prodecure but the you have to enter your
   sync key by hand.

4. If you have connected your clients, you can close the registration by running
   `weave-minimal` without the `--enable-registration` flag.

**Q:** Is this implementation standard compliant?  
**A:** Yes, it passes the official functional test suite (with [minor
       modifications][9]).

**Q:** Is it compatible with the latest version of Firefox?  
**A:** Not guaranteed. There is a new API draft, but not used in
       Firefox/Firefox ESR before 2014.

**Q:** Can I use a custom certificate for HTTPS?  
**A:** Yes, but import the CA or visit the url before you enable syncing.
       Firefox will show you a misleading error "invalid url" if you did not
       accept this cert before!
       If you are using Firefox on Android, you have to accept the certificate
       with the default Android Browser (called "Browser").
       Also [see here](#ssl-and-firefox-for-android) for
       information on a bug in Firefox for Android that might
       cause you troubles.

**Q:** It does not sync!  
**A:** Make sure, that `$ curl http://example.tld/prefix/user/1.0/example/node/weave`
       returns the correct sync url. Next, try to restart your browser. If that
       doesn't help, please file a bug report.

[9]: https://github.com/posativ/weave-minimal/issues/4#issuecomment-8268947

### Using a Custom Username

Mozilla assumes, you're using their services, therefore you can not enter a
non-valid email-address and Firefox will prevent you from doing this. But
there's an alternate way:

Instead of entering all your details into the first screen of "Firefox Sync
Setup" you click on "I already have a Firefox Sync Account". Before you can go
to the next step, you have to set up a user account in weave.

    $ weave-minimal --register bob:secret123
    [info] database for `bob` created at `.data/bob.c203011d1453ba7c`

Now you can continue your Firefox Sync Setup and click "I don't have the device
with me" and enter your username, password, "use custom server" -> url and
secret passphrase. That's all.


Webserver Configuration
-----------------------

### using lighttpd and mod_proxy

To run weave-minimal using [lighttpd][10] and mod_proxy you need to pass the
public url, e.g. `weave-minimal --base-url=http://example.org/sync ...`

    $HTTP["url"] =~ "^/weave/" {
        proxy.server = ("" =>
           (("host" => "127.0.0.1", "port" => 8080)))
        setenv.add-request-header  = ("X-Forwarded-Proto" => "https") # optionally for HTTPS
    }

[10]: http://www.lighttpd.net/

### nginx

    location ^~ /weave/ {
        proxy_set_header        Host $host;
        proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header        X-Forwarded-Proto $scheme;
        proxy_set_header        X-Script-Name /weave;
        proxy_pass              http://127.0.0.1:8080;
    }

### using apache and mod_proxy, with SSL

    <Location /sync>
        ProxyPass http://127.0.0.1:8080
        RequestHeader set X-Forwarded-Proto "https"
    </Location>

You can skip `RequestHeader`, if apache proxies the service on regular `http`.

### SSL and Firefox for Android

If Firefox for Android fails to sync properly, yet doesn't give any error
messages either, check out `adb logcat`. You might get an error similar to
this:

    javax.net.ssl.SSLPeerUnverifiedException: No peer certificate

That's because [Firefox for Android has a nasty
bug](https://bugzilla.mozilla.org/show_bug.cgi?id=756763) that doesn't allow
you to use self-signed certificates or certain ciphers.

For some reason, this error occurs not because your certificate is untrusted,
but because of ciphers. Adding RC4+RSA to the server's cipher list "fixes" that
issue.

For nginx the code to add would be:

    server {
        # ...
        # this makes ssl connections less secure!
        ssl_ciphers HIGH:!aNULL:!MD5:RC4+RSA;
        # ...
    }

Deployment
----------

For higher concurrency (if possible at all with SQLite), gevent will be used if
installed (`pip install gevent`). Furthermore, `weave-minimal` exports an
*application* object for uWSGI and Gunicorn, e.g.:

```bash
$ env ENABLE_REGISTRATION=1 DATA_DIR=/var/lib/... gunicorn weave -b localhost:1234
```

Do *not* use multiple processes to run `weave-minimal`. The code does not
acquire inter-process locks on the database and I have no plans to add an IPC
concurrency pattern to the rather simple code base (programmer's lame excuse,
I know).
