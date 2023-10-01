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
***Note 1:*** Set '<BLOCKIPS(BOOL)>' to 1 if you don't want your server to appear in common trackers such as tracker.killtube.org or gametracker. This will prevent your server from being duplicated two/three times on those tracker lists.

***Note 2:*** It is very likely you will need to set your net_ip to 0.0.0.0 or it won't work.

***Note 3:*** This will cause the player's IP address to return '127.0.0.1' when using getIP(). To fix this, you can utilize 'Info_ValueForKey(userinfo, "ip")' to get their real IP address. For example, edit 'ClientConnect' inside [g_client_mp.cpp](https://github.com/voron00/CoD2rev_Server/blob/master/src/game/g_client_mp.cpp) to [look like this](https://pastebin.com/mRWbrgi2) and edit 'gsc_player_getip' inside [gsc_player.cpp](https://github.com/voron00/CoD2rev_Server/blob/master/src/libcod/gsc_player.cpp) to [look like this](https://pastebin.com/hqmD1cxw).
\
\
\
Credits: [Libcod](https://github.com/kungfooman/libcod) for the rate limiter code.
\
\
\
Copyright (c) 2023, king-clan.com
All rights reserved.

This source code is licensed under the BSD-style license found in the
LICENSE file in the root directory of this source tree. 
