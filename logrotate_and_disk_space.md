## What Ate My Disk Space
One of the big culprits of rampant disk space consumption is log files. Linux has a built in process to handle rotation of logs called logrotate. Logrotate runs as a cron job, if you're interested this sits under /etc/cron.daily/logrotate. Cron just runs the scripts in this directory every day kicking off logrotate. 

When logrotate is run it will parse it's configuration, located at /etc/logrotate.conf as well as any files in the included directory under /etc/logrotate.d/ it will then evaluate the log files specified in these against the current state of the system and perform any actions needed. Logrotate will only run on files that are 'eligible' for rotation, that is to say they meet the criteria laid out in the configuration files. 

If log files are filling up a disk there's usually a couple of reasons: 

- Logrotate is broken, or is not configured to run against that file, and over time the file has grown very large.
- Logrotate is configured correctly however a recent change to an application or service has caused the speed at which the file grows to become much faster, since logrotate only runs daily if a log file grows substantially within a 24 hour period this could cause low disk space (an example might be a log file that has errors, and your application goes down, suddenly that log file starts getting written to repeatedly thus growing large in a matter of hours instead of its usual days). 
- The log file is being rotated, however the application that is accessing the file has kept a file descriptor to the file in use, and the kernel is not aware that the disk space can be reclaimed. 

### Find Those Logs
Before we can troubleshoot logrotate we will want ot find out what logs are causing the problem. I like to use a command like this to work my way down through directories 

```
du -xh ./* --max-depth=2 | grep G | sort -nr
```

You can switch the G out for M on smaller filesystems, otherwise it will only pull back results more than a GB in size. It's not perfect because it also pulls back any file names with the letters in them but it's a nice easy one to remember. You can amend the max-depth but I like to leave it at 2 and then traverse down into directories and run it again if needed. Usually this will pull back a couple of culprits. Maybe ossec, maybe an application log or syslog. Once we know what log is causing the issue we can start working out what the issue is. 

### Dates and Timestamps
Dates and timestamps will really help you when troubleshooting rotation. Once you know what file is causing the problem, you will want to start investigating what the logs for that application usually look like. Lets take a maillog as a good example. Here's the maillog on a system: 

```
root@pluto:/var/log# date
Thu Apr 19 16:50:01 UTC 2018
root@pluto:/var/log# ls -lah mail.log*
-rw-r----- 1 syslog adm    0 Jun 24  2015 mail.err
-rw-r----- 1 root   adm  29K Apr 19 16:01 mail.log
-rw-r----- 1 root   adm  68K Apr 19 06:00 mail.log.1
-rw-r----- 1 root   adm 5.8K Apr 18 06:03 mail.log.2.gz
-rw-r----- 1 root   adm 5.8K Apr 17 06:03 mail.log.3.gz
-rw-r----- 1 root   adm 5.8K Apr 16 06:00 mail.log.4.gz
-rw-r----- 1 root   adm 5.8K Apr 15 06:04 mail.log.5.gz
-rw-r----- 1 root   adm 6.4K Apr 14 06:09 mail.log.6.gz
-rw-r----- 1 root   adm 8.2K Apr 13 06:08 mail.log.7.gz
```

We can see that logrotate is working as we expect, if you look at the dates, timestamps and appended info we can see that every day this log is being rotated around the same time. What might this look like in a system having a problem? 

```
root@pluto:/var/log# date
Thu Apr 19 16:50:01 UTC 2018
root@pluto:/var/log# ls -lah mail*
-rw-r----- 1 syslog adm    0 Jun 24  2015 mail.err
-rw-r----- 1 root   adm 728M Apr 19 16:01 mail.log
-rw-r----- 1 root   adm  68K Apr 13 06:00 mail.log.1
-rw-r----- 1 root   adm 5.8K Apr 12 06:03 mail.log.2.gz
-rw-r----- 1 root   adm 5.8K Apr 11 06:03 mail.log.3.gz
-rw-r----- 1 root   adm 5.8K Apr 10 06:00 mail.log.4.gz
-rw-r----- 1 root   adm 5.8K Apr 09 06:04 mail.log.5.gz
-rw-r----- 1 root   adm 6.4K Apr 08 06:09 mail.log.6.gz
-rw-r----- 1 root   adm 8.2K Apr 07 06:08 mail.log.7.gz
```

Looking at the above we can see that the maillog is currently over 700MB in size, and if we look at the previous logs, it seems like this was rotated daily up until April 13th. Since then it doesn't seem to have been rotated. In this case we can be confident that the issue is with logrotation for the mail log. Let's follow that trail and have a look at the logrotate config for that.

#### Bad Config
Logrotate is neat in that it defines the logs to be rotated by absolute path, so you can usually just search for $log-file-name in the configs to see where that specific log is configured. Not much is configured in the top level /etc/logrotate.conf so let's check logrotate.d first

```
root@pluto:/etc# grep -R mail /etc/logrotate.d/
/etc/logrotate.d/rsyslog.disabled:/var/log/mail.info
/etc/logrotate.d/rsyslog.disabled:/var/log/mail.warn
/etc/logrotate.d/rsyslog.disabled:/var/log/mail.err
/etc/logrotate.d/rsyslog.disabled:/var/log/mail.log
```

That might be our problem, this file is called rsyslog.disabled, so logrotate isn't going to be parsing this. We would probably want to find out why that file had been modified or not replaced with an alternative. In the example we have above, the solution was to ensure we have the right rotation in place. In this case we could look at another server of a similar role to find the configuration it used for rotation and copy that across: 

```
root@mars:/etc/logrotate.d# grep -R mail /etc/logrotate.d/
/etc/logrotate.d/syslog-localweekly:"/var/log/mail.err" "/var/log/daemon.log" "/var/log/kern.log" "/var/log/auth.log" "/var/log/cron.log" "/var/log/messages" {
/etc/logrotate.d/syslog-localdaily:"/var/log/syslog" "/var/log/mail.log" {
/etc/logrotate.d/rsyslog.disabled:/var/log/mail.info
/etc/logrotate.d/rsyslog.disabled:/var/log/mail.warn
/etc/logrotate.d/rsyslog.disabled:/var/log/mail.err
/etc/logrotate.d/rsyslog.disabled:/var/log/mail.log
```

On that machine we have that syslog-localweekly file that looks like this: 

```
root@mars:/etc/logrotate.d# cat /etc/logrotate.d/syslog-localweekly
# This file was generated by Chef
# Do not modify this file by hand!
 
"/var/log/mail.err" "/var/log/daemon.log" "/var/log/kern.log" "/var/log/auth.log" "/var/log/cron.log" "/var/log/messages" {
  weekly
  rotate 4
  missingok
  notifempty
  delaycompress
  compress
  postrotate
    invoke-rc.d syslog-ng reload > /dev/null
  endscript
}
```

Next steps here might be to look at the history of changes on this server, and also to our chef configuration to see why that file might not have been applied. We could also look at the bash history and previous logins, particularly we'd want to look at changes made on or around the 13th which was the last known 'good' logrotation date. 

#### Rapidly Expanding Logfiles

Another scenario we see relatively frequently is when a log file begins to grow unusually fast. This means that even though logrotate might be working as expected, the file has grown so large in such a short span of time that we're still seeing the underlying filesystem filling up. There's a few ways we can identify this:

First, we'll want to check that logrotation looks normal. We can do that by referring to the same files that we did in the previous scenario and also by reviewing the logrotate logs themselves. In this scenario you'll usually see one, very large current log file. Let's use maillog as an example again:

```
root@pluto:/var/log# date
Thu Apr 19 16:50:01 UTC 2018
root@pluto:/var/log# ls -lah mail*
-rw-r----- 1 syslog sys    0 Jun 24  2015 mail.err
-rw-r----- 1 root   sys 2.2G Apr 19 16:01 mail.log
-rw-r----- 1 root   sys  68K Apr 18 06:00 mail.log.1
-rw-r----- 1 root   sys 5.8K Apr 17 06:03 mail.log.2.gz
-rw-r----- 1 root   sys 5.8K Apr 16 06:03 mail.log.3.gz
-rw-r----- 1 root   sys 5.8K Apr 15 06:00 mail.log.4.gz
-rw-r----- 1 root   sys 5.8K Apr 14 06:04 mail.log.5.gz
-rw-r----- 1 root   sys 6.4K Apr 13 06:09 mail.log.6.gz
-rw-r----- 1 root   sys 8.2K Apr 12 06:08 mail.log.7.gz
```

In this case, we can see this log file is very big. A handy way to see the span of the log is to run a head and a tail, with '-n1'. This will give you the first and last timestamps that logs were written (assuming that your log format actually captures timestamps, which i hope it does!):

```
head -n1 $LOGFILE
tail -n1 $LOGFILE
```
With a very busy logfile we might see that it is, for example, 1GB or more, but has only been written to since the morning and it's now 2pm. This implies that the log file is growing at a rapid rate and we should review some of the logs being written to find out why that is. It's important to investigate these as they are often symptomatic of a larger problem. In the case of maillogs, you might start to notice a large number of defferred mails in the logs. The alert you'd receive here might be for low disk space on '/var/log', but the actual issue could be something like the smtp server being down.

#### File Descriptors and Deleted Files
Sometimes you'll log into a server and see that 'df' reports a large amount of space utilised, but 'du' does not seem to match that output. For example: 

```
df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda1        20G   12G  6.8G  64% /
 
 
[root@pluto ~]# du -sh /
4G  /
```

This mismatch of reporting is due to the way that 'df' and 'du' work to report disk usage:

*df* measures against the primary superblock, with the key point here being that this includes open files (in memory, but not on disk). 
*du* actually reads the size of the files in real time as they are reporting. This is why df returns quickly and du returns more slowly. 


When a process needs to access a file, the kernel allocates a file descriptor. The process then interacts with this file descriptor and the kernel takes care of actually passing these requests through to the disk controller. Each process that needs to access a specific file will cause a new file descriptor to be allocated. A file descriptor is a pointer to the location of data on the disk.

Interestingly, there is not a one-to-one relationship between file descriptors and inodes. There is always one inode per file, but there may be multiple file descriptors that point to the same inode, and it's linked on disk location.

If an application is still writing to a log file after you've tried to delete it, the kernel will continue to store an entry for the file in its index until the application sends a 'close()' message to remove the file descriptor. In certain circumstances, this 'close()' signal doesn't get sent. Often, this will be because the application is poorly coded, or because the file it was writing to was deleted without the application knowing (sometimes if someone has manually rotated the file without running rotation postscripts etc.)

You can tell if this is the case by looking at what's returned with the following command: 

```
lsof | grep deleted
```
This will show you processes that have allocated file descriptors still open but that have been marked as deleted. There's two ways to 'fix' this. The neatest is to restart whatever application is keeping that file descriptor active.

There is a hacky way to actually free up the space without having to restart the application, but, generally, you should not do this. The cool thing about file descriptors is that you can use a similar method to restore the contents of a file that got accidentally deleted and still have file descriptors open. For example, if the socket file gets removed for a process, you can restore it from the same path. File descriptors can be found in 'lsof output', and you can use '/proc' to manipulate these. To essentially wipe any content out of that file with the descriptor still allocated, you can run the following where '$pid' is the process writing to the file and '$fd' is the number in the FD column from 'lsof'.

```
#echo " " > /proc/$pid/fd/$fd
```

Similarly, if you wanted to restore that file, you could do a:

```
#cp /proc/$pid/fd/$fd <your-restore-path>
```
