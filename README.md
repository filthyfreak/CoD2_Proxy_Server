# CoD2 Proxy Server
A proxy server for Call of Duty 2 written in C. Intended to work in conjunction with the '[CoD2rev_Server](https://github.com/voron00/CoD2rev_Server)' project.

[CoD2rev_Server](https://github.com/voron00/CoD2rev_Server) enables you to run a 1.0 server that accepts clients from all versions.

This program will make your 1.0 server appear when refreshing the server list whilst the client is on the 1.2 or 1.3 version.

The end result is a server that appears to be on 1.2/1.3 but is actually just a proxy that is redirecting traffic to/from your 1.0 server, allowing you to unify the entire player-base from across all versions.
\
\
\
Compile: ```gcc -o cod2proxy_lnxded cod2proxy_lnxded.c```

Usage: ```cod2proxy_lnxded <FORWARD_TO> <LISTEN_ON> <SHORTVERSION> <FS_GAME> <BLOCKIPS(BOOL)>```

Example: ```cod2proxy_lnxded 28960 28990 1.3 kingbot 1```
\
\
\
Note 1: Set '<BLOCKIPS(BOOL)>' to 1 if you don't want your server to appear in common trackers such as tracker.killtube.org or gametracker. This will prevent your server from being duplicated two/three times on those tracker lists.

Note 2: It is very likely you will need to set your net_ip to 0.0.0.0 or it won't work.
\
\
\
Credits: Voron for the rate limiter code.

Copyright (c) 2023, king-clan.com
All rights reserved.

This source code is licensed under the BSD-style license found in the
LICENSE file in the root directory of this source tree. 
