### What Ate My Disk Space
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

### Parsing Config
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

