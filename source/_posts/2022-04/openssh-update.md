---
title: openssh update
date: 2022-04-21 17:30:28
updated: 2022-11-03 15:42:55
tags:
- Linux
- OpenSSH
categories: Text-only
---
Another month, another update. This time, OpenSSH decided to add a key-exchange
algorithm that just so happens to break my GPG+YubiKey ssh setup.

Said kex algorithm is `sntrup761x25519-sha512@openssh.com`. If you don't remove
it, any attempts to connect to the server with the "current" yubikey (I have a
5.2.7 yubikey) will result in the scdaemon spitting a not-at-all helpful error
message at you: 

```
gpg-agent[1074]: scdaemon[1074]: app_auth failed: Invalid value
```

This was a bit confusing and took me a while to figure out why it happens[^2], even
with gpg-agent set to output all information possible.

Solution[^1]:
```diff
# /etc/ssh/sshd_config
+ KexAlgorithms -sntrup761x25519-sha512@openssh.com
```

It's worth noting that this config only applies to sshd. If you're a client trying
to connect to a remote server that doesn't have this, it will not work. I'm not sure
if this is patched yet.

[^1]: https://bugs.archlinux.org/task/74423
[^2]: https://lwn.net/Articles/890803/
