---
layout: post
title: "Setup MAMP Environment on OS X 10.8"
date: 2013-03-04
tags: OS X, PHP, Apache, MySQL
description: "This article describes the procedure of setting up a multi-host Mac-Apache-MySQL-PHP (MAMP) development environment from scratch on OS X Mountain Lion."
---

This post is a tutorial of how to set up MAMP (Mac-Apache-MySQL-PHP) development environment on Mac OS X 10.8 (a.k.a Mountain Lion).

OS X is a developer-friendly operating system with a bunch of ready-to-use tools and components (though many are not the latest and some times [even unusable](sed-twitter)).
Although there is [bundled MAMP](mamp) available.
Mountain Lion is shipped with Apache 2.2 and PHP 5.3.15, and with these two you can set up MAMP (relatively) easily (the only missing part is MySQL).

A little tip: almost all commands below **requires administrator privilege**.
So make sure you are among the `sudoers`.

## Set up Apache with Virtual Hosts

Before Mountain Lion, you can enable OS X builtin web server [via System Preference](webserver-sharing).
However, with ML, you have to enable it via command line.

You can type command `apachectl start` to use it right now.
Visit `http://localhost` in your browser and if you see "It works!"  then it really works! :)

The default `DocumentRoot` is at `/Library/WebServer/Documents`.
You can host static pages there but we certainly need more: the ability to execute PHP scripts and virtual hosts.

Let's do it.

First use your favorite text editor to edit `/etc/apache2/httpd.conf`.
Uncomment `line 117` (you can search `php` to locate it):

```
LoadModule php5_module libexec/apache2/libphp5.so
```

and `line 477` (search `vhosts`):

```
Include /private/etc/apache2/extra/httpd-vhosts.conf
```

Then edit `extra/httpd-vhosts.conf` to add your virtual hosts.
You can follow the two existing sample vhost configurations. (<abbr title="By the way">BTW</abbr>, I recommend you to comment the examples out.)


A sample configuration is as follows:

```
<VirtualHost *:80>
    ServerAdmin you@example.com
    DocumentRoot "/Users/you/Sites/sample"
    ServerName example.local
    ErrorLog "/private/var/log/apache2/example.local-error_log"
    CustomLog "/private/var/log/apache2/example.local-access_log" common
    <Directory "/Users/you/Sites/sample">
        AllowOverride FileInfo
        Allow from all
    </Directory>
</VirtualHost>
```

Note the `Directory` section.
The `AllowOverride` directive enables `.htaccess` at `DocumentRoot`, which is useful if you want to rewrite URL to `index.php` (a must-have if you do WordPress development like me).
The `Allow` directive ensures visitors from all hosts can access the server.

Now you can type `httpd -S` command to test if anything is going well.
If you see `Syntax OK` on the last line outputed (possibily along with some warnings), you are cool.
Type `apachectl restart` to **restart Apache**.
You can test if PHP works by writing a simple script in the doc root and visiting it.

## Install and Configure MySQL

As said above, you need to download MySQL from the [official website](mysql-download) first.
The current latest version supports 10.7 and by my experience, it works fine on 10.8.
If you are as lazy as me, you'd prefer `.dmg` as it is simple to install.

After you mount the disk image, you will see 4 items (3 installable plus a readme).
Theoretically only the first one (MySQL) is required.
The second item (preference panel) is recommended as with it you can directly start/stop MySQL in the System Preference.
Whether to install third one (startup item) is up to you.

**Attention**: there is a catch after MySQL installation. The PHP function `mysql_connect` requires the existence of `/var/mysql/mysql.sock`.
However, the socket file is actually installed at `/tmp/mysql.sock`.
I have no idea if everyone would encounter this, but if you do, make a symbolic link like this:

```
mkdir -p /var/mysql
ln -s /tmp/mysql.sock /var/mysql/mysql.sock
```

Now start MySQL and you can test it the way you prefer.

---

Now your MAMP environment should work smoothly.

I wrote this because I switched to a new MacBook and had to set up a development environment for WordPress ***again***.
Having struggled with the same hazards a second time, I think I'd better write this little tutorial for everyone who needs it, including (maybe) the future me.

[sed-twitter]: https://twitter.com/kavinyao/status/288582100930662400
[mamp]: http://sourceforge.net/projects/mamp/
[webserver-sharing]: http://macs.about.com/od/networking/qt/websharing.htm
[mysql-download]: http://www.mysql.com/downloads/mysql/
