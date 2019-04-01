# Limit system dump output

## Problem statement
If the SONiC system is running for quite some time it may have a huge amount of logs which generate_dump.sh script will collect in a single tar ball which has size of order of several hundreds of Mb.

Since commonly there is no need to gater all logs but just logs for last several days this design doc introduces a new options for ```show techsupport``` command

## Solution
## Introducing option to specify how much logs you want to dump

```
admin@sonic:~$ show techsupport -h
Usage: show techsupport [OPTIONS]

  Gather information for troubleshooting

Options:
  --since         Collect logs since given date
                  a human readable string that represents a date
                  e.g:
                    "a day ago",
                    "3 days 5 hours ago"
                    "24 March"
  --verbose       Enable verbose output
  -?, -h, --help  Show this message and exit.
```

```
admin@sonic:~$ show techsupport --since '1 day ago'
```

Logrotation in SONiC is configured that it preserves modification time of syslog.$n.gz and this time matches the time in last written log message in this file

## generate_dump.sh flow
1. Disable logrotatation before collecting logs<br>As it may mess with generate_dump.sh
2. If there is "--since" parameter only collect logs from /var/log that are newer than "since" argument
3. Default behaviour remains the same meaning --last default is infinity
4. Enable logrotation back

## Open questions
