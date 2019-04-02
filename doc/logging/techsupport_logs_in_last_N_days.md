# Limit system dump output

## Problem statement
If the SONiC system is running for quite some time it may have a huge amount of logs and core file which generate_dump.sh script will collect in a single tar ball which has size of order of several hundreds of Mb.

Since commonly there is no need to gater all logs but just logs and core file for last several hours/days this design doc introduces a new options for ```show techsupport``` command

## Solution
## Introducing option to specify how much logs you want to dump

```
admin@sonic:~$ show techsupport -h
Usage: show techsupport [OPTIONS]

  Gather information for troubleshooting

Options:
  --since DATE    Collect logs and core files since given DATE
                  a human readable string that represents a date
                  e.g:
                    "yesterday"
                    "2 pm"
                    "24 March"
  --verbose       Enable verbose output
  -?, -h, --help  Show this message and exit.
```

- new option for generate_dump.sh script and validation with date utility:<br>
```bash
date --date="${SINCE_DATE}" > /dev/null || abort "Invalid date expression passed"
```

## Examples

```
admin@sonic:~$ show techsupport --since=yesterday
```

```
admin@sonic:~$ show techsupport --since='24 march'
```

Logrotation in SONiC is configured that it preserves modification time of syslog.$n.gz and this time matches the time in last written log message in this file

## generate_dump.sh flow
* Disable logrotatation before collecting logs<br>As it may mess with generate_dump.sh
* If there is "--since" parameter only collect logs from /var/log that are newer than "since" argument
  * Create a temporary reference file, e.g:<br>```bash touch --date="${DATE}" /tmp/datetime_reference```
  * Find logs, core files passing --newer parameter to find utility:<br>```bash find -L /var/log -type f --newer /tmp/datetime_reference```
* Default behaviour remains the same meaning "--since" default is infinity
* Enable logrotation back

## Exclude /etc/mlnx, /etc/mft/

## Open questions
