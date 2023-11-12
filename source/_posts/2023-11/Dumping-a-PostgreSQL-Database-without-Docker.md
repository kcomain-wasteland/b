---
title: Dumping a PostgreSQL Database without Docker
date: 2023-11-12 14:02:43
updated: 2023-11-12 19:15:40
tags:
- PostgreSQL
- Nix
- Docker
- Guide
- Linux
categories: Text-only
---

When migrating systems, you might have been hit with the problem of migrating a PostgreSQL database from a read-only filesystem snapshot.
If you think this sounds like a really specific skill issue, I can assure you that it absolutely is. Anyways, since the switch to NixOS for my main server in August, there were a bunch of docker volumes in my filesystem snapshot, some are psql databases. I don't have docker/podman installed, so how am I going to export a dump of the databases?
Well, welcome to this post.

Let's rule out things that won't work. 
1. copying the entire directory and paste it right into the main psql database

and that's about all the things that I can think about.

For the purposes of this guide, we will assume the following variables:
```bash
VOLUME=/directory/to/the/docker/volume
```

# Get the database out of a volume

First and foremost, we need to get the database directory out of the volume first. Since the filesystem is read-only, we have no choice but to move it out. You can alternatively do some cool insane stuff like mounting an [overlayfs](https://docs.kernel.org/filesystems/overlayfs.html) on the volumes directory, but for the purpose of this guide we are not going to do that to keep this simple.
We can simply copy the entire database to an empty temporary database:
```ShellSession
$ TMPDIR=$(mktemp -d)
$ cp $VOLUME/_data $TMPDIR -r  # in postgres:15-alpine, the data dir is _data inside the volume. change it accordingly if it is different for you.
```

Now that we have the data dir, change the ownership of `$TMPDIR` recursively to a user that's not `root`, as pg_ctl refuses to run as root.
```ShellSession
$ sudo -s 
# chown -R postgres:postgres $TMPDIR
# #        ^~~~~~~~~~~~~~~~~ we use postgres because the original db is ran with postgres. I have not tested anything else.
```

# Set some config so postgres doesn't explode

We then have to replace the port of our working shadow server, so as to not conflict with the main psql database. You can skip this part if you don't have one running.
```ShellSession
$ sed -i "s/^#\?listen_addresses.*$/listen_addresses = '' # disable listening on tcp ports/" _data/postgresql.conf
$ sed -i "s/^#\?\(port =\).*$/\1 31280/" ./_data/postgresql.conf
$ #                              ^^^^^ this can be any port that's not the main psql port.
```

Additionally, harden the access configuration for the peace of mind.
```ShellSession
$ cat > _data/pg_hba.conf <<EOF
local all all trust
EOF
```

# Run the server
Now we are finally ready to run the server. As explained above, we cannot run the database server as root.

```ShellSession
# alias pg_run="sudo -u postgres"
# pg_run pg_ctl -D _data start
```

# Dump the database
Chances are the default user (usually the only user) in the database isn't `postgres`. If you use docker compose, just figure it out from your compose.yaml.
I unfortunately cannot help you at the moment if you don't know the database and/or user.

Afterwards, you can simply do the following to get a db dump.
```ShellSession
$ pg_run pg_dump -p 31280 -U [user] [database] -s  | nix run n#pv > [database].schema.dump.sql
$ pg_run pg_dump -p 31280 -U [user] [database] -ab | nix run n#pv > [database].data.dump.sql
```

The dump is split into 2 files for easier checking, and easier editing as the owner role might not be the same.
If that is the case, simply change all the old owners in the `schema.dump.sql` to the new one.

```ShellSession
$ sed -i 's/\(OWNER TO\) .*\;$/\1 "[final-user]" ;/' [database].schema.dump.sql
```

# Dump the dump to the data dumpster (database)
This part is easy. Maybe. 

Up to this point, in no commands have we told psql to dump users and the databases. We will now have to create them before loading the database dumps.

```ShellSession
$ # create the user
$ pg_run createuser -eDlPRS [user]
$ # create the database
$ pg_run createdb -eO [user] [database_name]
```

We can now import the dump.

```ShellSession
$ pg_run psql -1 -v ON_ERROR_STOP=1 -d [database] -f [database].schema.dump.sql # load the schema first
$ pg_run psql -1 -v ON_ERROR_STOP=1 -d [database] -f [database].data.dump.sql
```

At this point we should have a fully recovered database. We can then move on to removing the temporary files.

# Removing the dump
First, pull the plug on the shadow server.

```ShellSession
$ pg_run pg_ctl -D _data stop
waiting for server to shut down.... done
server stopped
```

Now, just remove the temporary directory with the same method you normally use.

```ShellSession
$ nix run n#srm $TMPDIR -- -RxCvv
```

That's all!
