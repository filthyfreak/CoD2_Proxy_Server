# CoD2_Proxy_Server
A proxy server for Call of Duty 2 written in C. Designed to work in conjunction with the 'CoD2rev_Server' project.

```Compile command: gcc -o cod2proxy_lnxded cod2proxy_lnxded.c```

Usage: ```cod2proxy_lnxded <FORWARD_TO> <LISTEN_ON> <SHORTVERSION> <FS_GAME> <BLOCKIPS(BOOL)>```

Example: ```cod2proxy_lnxded 28960 28990 1.3 kingbot 0```

Note: Set '<BLOCKIPS(BOOL)>' to 1 if you don't want your server to appear in common trackers such as tracker.killtube.org or gametracker. This will prevent the same server appearing two/three times in the tracker lists.

Credits: Voron for the rate limiter code.

Copyright (c) 2023, king-clan.com
All rights reserved.

This source code is licensed under the BSD-style license found in the
LICENSE file in the root directory of this source tree. 
