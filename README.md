# apache-smemstat

Use smem to print an accurate 1-line summary about memory usage of apache httpd processes

### Why?

Unfortunately, none of the standard tools in Linux (`top`, `ps`, `pidstat`) currently have the ability to print unique RAM usage per process; instead, they always print RSS which includes shared memory, like shared libraries used by multiple processes (e.g., libc). That means that if you sum up the RSS of multiple processes on your system, you'll be over-counting.  

For processes like Apache httpd and Google Chrome that fork tons of times to serve their users, this is even more dramatic. On my system right now, the total RSS sum of all Google Chrome processes is 876 MiB; however, when I take shared memory into account, I can see that it's actually only 394 MiB.

When calculating MPM parameters for Apache httpd, it's pretty important to get accurate data of the current state of things. An unloaded httpd service on my RHEL 6 VM configured with `MinSpareServers 120` uses 294 MiB (2.4 MiB/process) -- if I look at RSS. When I take shared memory into account, it's clear that each (unloaded) process is only using 148 KiB, and then a bit is shared across all of them, for an accurate total usage of 19.4 MiB.

### How?

While none of the standard tools show this information, it is thankfully made available by the Linux kernel via /proc. A couple 3rd-party tools (`smem` and `smemstat`) process this information beautifully. (Both are available in Fedora & EPEL.)

Example output:

```
# smem -P ^/usr/sbin/httpd -tk
  PID User     Command                         Swap      USS      PSS      RSS 
14621 apache   /usr/sbin/httpd                    0   152.0K   292.0K     2.4M 
14622 apache   /usr/sbin/httpd                    0   152.0K   292.0K     2.4M 
14623 apache   /usr/sbin/httpd                    0   152.0K   292.0K     2.4M 
14624 apache   /usr/sbin/httpd                    0   152.0K   292.0K     2.4M 
14625 apache   /usr/sbin/httpd                    0   152.0K   292.0K     2.4M 
14626 apache   /usr/sbin/httpd                    0   152.0K   292.0K     2.4M 
14627 apache   /usr/sbin/httpd                    0   152.0K   292.0K     2.4M 
14628 apache   /usr/sbin/httpd                    0   152.0K   292.0K     2.4M 
14618 root     /usr/sbin/httpd                    0     1.1M     1.3M     3.8M 
-------------------------------------------------------------------------------
    9 2                                           0     2.3M     3.5M    23.2M 
    
# smemstat -p httpd
  PID       Swap       USS       PSS       RSS User       Command
 14618     0.0 B  1092.0 K  1284.0 K  3940.0 K root       /usr/sbin/httpd
 14628     0.0 B   152.0 K   292.0 K  2476.0 K apache     /usr/sbin/httpd
 14627     0.0 B   152.0 K   292.0 K  2476.0 K apache     /usr/sbin/httpd
 14626     0.0 B   152.0 K   292.0 K  2476.0 K apache     /usr/sbin/httpd
 14625     0.0 B   152.0 K   292.0 K  2476.0 K apache     /usr/sbin/httpd
 14624     0.0 B   152.0 K   292.0 K  2476.0 K apache     /usr/sbin/httpd
 14623     0.0 B   152.0 K   292.0 K  2476.0 K apache     /usr/sbin/httpd
 14622     0.0 B   152.0 K   292.0 K  2476.0 K apache     /usr/sbin/httpd
 14621     0.0 B   152.0 K   292.0 K  2476.0 K apache     /usr/sbin/httpd
Total:     0.0 B  2308.0 K  3620.0 K    23.2 M
```

In the above output, the RSS column is the same one reported by standard tools like `top` and `ps`.

The USS column shows the *Unique Set Size* -- i.e., unshared memory specific to that process. This is the best indicator of how much RAM a particular process is using.

The PSS column shows the *Proportional Set Size* -- i.e., USS + an appropriate proportion of the shared memory. This is usually the best way to gauge the total RAM usage of a group of processes (like all `httpd`).

### Wait, so what's apache-smemstat?

This is is a way to quickly summarize smemstat-like information about all httpd processes into a single column-separated line -- e.g., for writing out to a log file regularly.

Here's what the help page looks like, including some examples.

```
$ apache-smemstat -h
Usage: apache-smemstat [-t|--timestamp]
Print a 1-line summary about memory usage of apache httpd processes

The smem command is required in order to take shared memory into account. In
RHEL, the smem package is available from EPEL.

Example usage:
  
  # apache-smemstat
  apache httpd |  120 total PIDs |   3.9 MiB/each avg |   464.5 MiB total

Switch out the "apache httpd" for a timestamp with an optional -t:
  
  # apache-smemstat -t
  2015-11-19 23:40:39 |  120 total PIDs |   3.9 MiB/each avg |   464.5 MiB total

Or customize the "header" environment variable:
  
  # header=1448019780 apache-smemstat
  1448000502 |  120 total PIDs |   3.9 MiB/each avg |   464.5 MiB total

Note that the environment variables "pFilter" (process filter) and "uFilter"
(user filter) can be tweaked to change behavior altogether -- e.g., to watch
processes other than httpd. Each is passed as an argument to smem's -P and -U
options (respectively) -- where they will be treated as regular expressions.
Defaults:
  
  pFilter="^/usr/sbin/httpd"
  uFilter="^apache$"

If pFilter is modified and uFilter is not, then uFilter will be reset to match
the current user. Examples:
  
  $ pFilter=^bash apache-smemstat
  Warning: running as non-root; only able to inspect own (rsaw) processes
  ^bash |    3 total PIDs |   3.6 MiB/each avg |    11.5 MiB total
  
  $ pFilter=^/opt/google/chrome/chrome apache-smemstat -t 2>/dev/null
  2015-11-20 00:47:26 |   12 total PIDs |  25.2 MiB/each avg |   391.8 MiB total
  
  # header="login screens" pFilter='^/sbin/(a|min)getty' apache-smemstat -t
  login screens |    6 total PIDs |   0.1 MiB/each avg |     0.5 MiB total

See also related utilities: smem, smemstat, pidstat

Version info: apache-smemstat v0.1.0 last mod 2015/11/20
See <github.com/ryran/apache-smemstat> to report bugs/suggestions or kudos
```

### Questions? Comments?

I'd love to hear from you. [File an issue.](https://github.com/ryran/apache-smemstat/issues)
