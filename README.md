Per Route Metrics and TCP Window Scaling
========================================

Bug in 4.4 BSD based TCP implementations

## Description

The TCP window scale option is incorrectly set when using a `recvpipe`
per route metric value greater than 65535 (the maximum size of the 
TCP window size field in the TCP header).  Before I get into the
details of the problem, here is a little background for those not
familiar with TCP window sizes, window scale option, and recvpipe
metric. 

The `recvpipe` per route metric enables an administrator to change the
size of the receive socket buffer on a per route basis.  Why would you
want to do this?  Because the size of the receive socket buffer is 
the size of the TCP window that your host advertises.  This allows
an administrator to set the TCP window size without having to rebuild
an application using setsockopt and RCVBUF, or without having to change
the size on a global basis via sysctl -w net.inet.tcp.recvspace.

The `recvpipe` option is great when you only need to change the 
window size for a particular route such as a long, fat, satellite
link.  See Stevens TCP/IP Illustrated v1 section 20.7 Bulk Data
Throughput - Bandwidth Delay Product for more information on the 
use of TCP window sizes.

The window scale option is exchanged ONLY on the first SYN packet with 
another host.  It must be used when the sender wishes to advertise
a TCP window size greater than 65535 because the size of the window
size field in the TCP header is only a 16bit value.  The window
scale option is a value that the receiving host should left shift
the received window size.  This allows a host to advertise a maximum
window size of 1,073,725,440 (65535x2^14).  See Stevens TCP/IP 
Illustrated v1 section 24.4 Window Scale Option for more details. 
	
It appears that almost all of the 4.4BSD derived TCP/IP stacks 
exhibit this bug.  In the case of OpenBSD, when `connect()` is called, 
`tcp_usrreq()` calculates the size of the window scale option using
`so->so_recv.sb_hiwatt`, which at this point, is currently set to 
the system wide default, 16384, or to another value that was set
via `sysctl` or `setsockopt()`. The result of the window scale calculation
is stored in `tp->request_r_scale`. 

`tcp_output()` is then called to send the SYN packet, which in turns 
calls `tcp_mss()`.  `tcp_mss()` checks to see if any per route `recvpipe`
metrics exist.  If the metric exists, the socket buffer is changed
to the appropriate size via sbreserve.  Unfortunately, the size of
the socket has just changed, but no recalculation of the window 
scale has been performed.

After `tcp_output()` calls `tcp_mss()`, it finally gets to the point
where it appends the tcp window scale option to the packet using
the value that was stored in `tp->request_r_scale`, which has been
unchanged since it was originally set in `tcp_usrreq()`.  This is a	
problem because the socket buffer size could have been changed to 
a value significantly larger via a per route metric which would 
require a new calculation of the window scale.
	
## How To Repeat

Add a route to a network inside of your routing table with a 
recvpipe option that would require a window scale option to be
set.  In this example, I use 102400 which is a value greater than
the 65535 window size field.  This would result in a window scale
option to be calculated to 1.

```
	route add -net 172.25.1.0 10.10.10.1 -recvpipe 102400
```

Start tcpdump to that network, and then telnet to a host on that
network.  Examine the first SYN packet sent by your host and 
look at the window scale option.  It will be set to ZERO!!  This
is incorrect. It should have been 1. 

Note, if you had changed your system wide default socket buffer 
recvspace via sysctl to this same value, 102400, and repeated the
test, the window scale is calculated correctly.  It is only 
incorrect when using the recvpipe option.
	
## Fix

I've tested this patch on my 2.6 box and it works properly. If I 
repeat the test above, it results in the proper window scale. It
has not been tested on 2.7 even though I've created a patch off 
of the latest sources. However, the buggy code is the same in all
versions of OpenBSD.

```
*** tcp_input.c.v1.66   Sun Jul  9 07:28:43 2000
--- tcp_input.c Sun Jul  9 07:31:53 2000
***************
*** 2914,2919 ****
--- 2914,2926 ----
  		if (bufsize > sb_max)
  			bufsize = sb_max;
  		(void)sbreserve(&so->so_rcv, bufsize);
+ #ifdef RTV_RPIPE
+ 		if (rt->rt_rmx.rmx_recvpipe > 0) {
+ 			tp->request_r_scale = 0;
+ 			while (tp->request_r_scale < TCP_MAX_WINSHIFT &&
+ 			      (TCP_MAXWIN << tp->request_r_scale) < so->so_rcv.sb_hiwat)
+ 			  tp->request_r_scale++;
+     }      
+ #endif
  	}
  	tp->snd_cwnd = mss;
```

## GNATS Bug Information

This was filed back in 2000.

```
>Number:         1310
>Category:       kernel
>Synopsis:       TCP window scale incorrectly set when 'recvpipe' per route metric is used
>Confidential:   no
>Severity:       non-critical
>Priority:       low
>Responsible:    bugs
>State:          open
>Class:          sw-bug
>Submitter-Id:   net
>Arrival-Date:   Sun Jul  9 13:10:01 MDT 2000
>Last-Modified:
>Originator:     Pete Kazmier
>Organization:
net
>Release:        All OpenBSD releases
>Environment:
  
  System      : OpenBSD 2.6
	Architecture: OpenBSD.i386
	Machine     : i386
```
