---
lang: en-US
title: A trip to Heroku
tag:
  - tech
---

# Deployment hell, with proprietary software
---

*May 26, 2021*

### Deploying on Heroku is such an easy task with tutorials are you joking?

Well, technically yes, but that's the case when you are using a non-proprietary database.
MSSQL addon on Heroku charged 5$/month, which is a problem with undergraduates who are not
financial independence. In my case, I asked my supervisor to provide me a server and
connects with SQL authentication and this solves the pricing problem. However, this server
was quite old (Microsoft SQL Server 2012 - 11.0.5058.0), and that was the root of all
troubles. I'm not having such privilege to upgrade the server, so I have to work around
with what I have.

### Everything starts with good vibe...

Before deploying, I tried to set up a local page to check the settings. Everything was
perfect! Then I install Heroku CLI, start adding buildpacks and again host local web using
Heroku. Everything looks great! With all my confidence, I pushed to Heroku's git and
deployed the web. Yay, landing page looks cool. Not and all navigations behaves correctly.
Not until I tried to intefere with the database when it responded:

![Internal Server Error](/assets/img/deployment-hell/500.png)

Well, of couse I expect some missteps from myself since this is my first time deploying.
However, this is still strange since I tried host local web using Heroku's config vars and
nothing went wrong. I reviewed the log, the error was:

```console
'01000', "[01000] [unixODBC][Driver Manager]Can't open lib 'ODBC Driver 17 for SQL Server' : file not found (0) (SQLDriverConnect)"
```

Hmmm, let's run `odbcinst -j` to see what happened:

```console
unixODBC 2.3.6
DRIVERS............: /etc/odbcinst.ini
SYSTEM DATA SOURCES: /etc/odbc.ini
FILE DATA SOURCES..: /etc/ODBCDataSources
USER DATA SOURCES..: /app/.odbc.ini
SQLULEN Size.......: 8
SQLLEN Size........: 8
SQLSETPOSIROW Size.: 8
```

When I checked `/etc/`, there was no `odbcinst.ini`. Surely the Dyno could not find it and
fallback to default setting. Additionally, I forgot to install `msodbc`. Having looked
around for solutions, I bumped into a true gold conversation and thanks to
[@boboldehampsink's suggestion](https://github.com/heroku/heroku-buildpack-php/issues/417#issuecomment-760696609)
that I could finally solve this problem. Although I recommend looking at his comment, here
are the steps to reproduce:

1. Add `apt` buildpacks
2. Create an `Aptfile`, which contains these following (currently updated with Microsoft)

```console
https://packages.microsoft.com/ubuntu/20.04/prod/pool/main/u/unixodbc/libodbc1_2.3.7_amd64.deb
https://packages.microsoft.com/ubuntu/20.04/prod/pool/main/u/unixodbc/odbcinst_2.3.7_amd64.deb
https://packages.microsoft.com/ubuntu/20.04/prod/pool/main/u/unixodbc/odbcinst1debian2_2.3.7_amd64.deb
https://packages.microsoft.com/ubuntu/20.04/prod/pool/main/u/unixodbc/unixodbc_2.3.7_amd64.deb
https://packages.microsoft.com/ubuntu/20.04/prod/pool/main/u/unixodbc/unixodbc-dev_2.3.7_amd64.deb
https://packages.microsoft.com/ubuntu/20.04/prod/pool/main/m/msodbcsql17/msodbcsql17_17.6.1.1-1_amd64.deb
https://packages.microsoft.com/ubuntu/20.04/prod/pool/main/m/mssql-tools/mssql-tools_17.6.1.1-1_amd64.deb
```

3. Create an `odbcinst.ini` file at `/app`

```console
[ODBC Driver 17 for SQL Server]
Description=Microsoft ODBC Driver 17 for SQL Server
Driver=/app/.apt/opt/microsoft/msodbcsql17/lib64/libmsodbcsql-17.6.so.1.1
UsageCount=1
```

4. Set up environment variables by:

```console
heroku config:set ACCEPT_EULA="Y"
heroku config:set ODBCSYSINI="/app"
```

Being done with those, you should get your settings like this

**Input**
```console
odbcinst -j
```

**Output**
```console
unixODBC 2.3.7
DRIVERS............: /app/odbcinst.ini
SYSTEM DATA SOURCES: /app/odbc.ini
FILE DATA SOURCES..: /app/ODBCDataSources
USER DATA SOURCES..: /app/.odbc.ini
SQLULEN Size.......: 8
SQLLEN Size........: 8
SQLSETPOSIROW Size.: 8
```

**Input**
```console
cat /app/odbcinst.ini
```

**Output**
```console
[ODBC Driver 17 for SQL Server]
Description=Microsoft ODBC Driver 17 for SQL Server
Driver=/app/.apt/opt/microsoft/msodbcsql17/lib64/libmsodbcsql-17.6.so.1.1
UsageCount=1
```

**Input**
```console
ls /app/.apt/opt/microsoft/msodbcsql17/lib64/
```

**Output**
```console
libmsodbcsql-17.6.so.1.1
```

### Behind a problem, is another problem...

I've managed to properly configure driver paths, yet the new problem was:

```console
'08001', '[08001] [Microsoft][ODBC Driver 17 for SQL Server]SSL Provider: [error:1425F102:SSL routines:ssl_choose_client_version:unsupported protocol] (-1) (SQLDriverConnect)'
```

Errr... what? SSL problem? I had again look for various forums. The problem was the
minimum protocol of Ubuntu 20.04 being `TLSv1.2`, while old server use `TLSv1.0` or
`TLSv1.1.` All answers were to modify `/etc/ssl/openssl.conf` and to set
`MinProtocol=TLSv1.0`. However, since I'm on remote shell, not root, I did not have
privilege to modify such file.

![Not root](/assets/img/deployment-hell/i-am-not-groot.webp)

Besides, my local machine could connect to the server with out changing any
configuration. I rechecked OpenSSL version, and it turned out the version on
Dyno was 1.1.1f compared to 1.1.1k on my machine. Thus I decided to update OpenSSL
[via a buildpack](https://github.com/9a24f0/heroku-buildpack-openssl). Thanks to
[@Ben Last](https://github.com/benlast) with his excellent work that I could upgrade
the version with ease.

The last work was to reconfigure `LD_LIBRARY_PATH` so that two versions not conflict
with each other. I would rather modify the buildpack to achieve correct path right
away, though seemingly the dyno wiped off `tmp/` during deployment. As we all know,
it is what it is, and here are steps to work around:

1. On dyno, run the following command and copy the correct path
```console
export LD_LIBRARY_PATH=~/openssl/lib${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH} && echo $LD_LIBRARY_PATH
```
2. After that, simply set Heroku's environment variable as
```console
heroku config:set LD_LIBRARY_PATH=your-copied-path
```

After this, we achieve such thing as:

**Input**

```console
openssl version
```

**Output**

```console
OpenSSL 1.1.1f 31 Mar 2020 (Library: OpenSSL 1.1.1k 25 Mar 2021)
```

### Conclusion
So what have I learnt from this? First thing is how to deploy on a cloud VM, where we
were given the resource but not the privilege. Seriously, the problem was not so
complicated as such if I could have a VM to install everything the same as my machine.
But hey, I've learnt how to work around with such limited permission, so better give
a thanks for the situation.

Secondly, you should have your system updated. As of currently available Ubuntu 20.04
and 20.10, the OpenSSL version is still `1.1.1f`, which could not connect to SQL
Server 12. Normal company would have a life span of more than 10 years with a database
not upgraded. The version that I provided was release on June 2014, and it's not yet 10
years since then. Upgrade to Ubuntu 21 provides you with `1.1.1j`, which solves the
problem directly and without any configuration.

Last but not least, if you could somehow have permission to upgrade the old database,
do it rightaway. I saw many questions about how to connects to a 2008 SQL Server,
which is officially unsupported by Microsoft now. The 2012 version is somehow under
support since the extended date would be July 2022. Yeah I believe my opinion about
upgrading the database will be accepted soon, but we will save it for a rainy day.