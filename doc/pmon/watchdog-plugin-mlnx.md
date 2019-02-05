# Watchdog plugin on Mellanox platforms design #

### Rev 0.1 ###

### Revision ###

 | Rev |     Date    |       Author       | Change Description                |
 |:---:|:-----------:|:------------------:|-----------------------------------|
 | 0.1 |  01/21/2019           |      Stepan Blyshchak      | Initial version        |
 
### Requirements ###

```python
def arm(self, seconds):
        """
        Arm the hardware watchdog with a timeout of <seconds> seconds.
        If the watchdog is currently armed, calling this function will
        simply reset the timer to the provided value. If the underlying
        hardware does not support the value provided in <seconds>, this
        method should arm the watchdog with the *next greater* available
        value.
        Returns:
            An integer specifying the *actual* number of seconds the watchdog
            was armed with. On failure returns -1.
        """
    def disarm(self):
        """
        Disarm the hardware watchdog
        Returns:
            A boolean, True if watchdog is disarmed successfully, False if not
        """

    def is_armed(self):
        """
        Retrieves the armed state of the hardware watchdog.
        Returns:
            A boolean, True if watchdog is armed, False if not
        """

    def get_remaining_time(self):
        """
        If the watchdog is armed, retrieve the number of seconds remaining on
        the watchdog timer
        Returns:
            An integer specifying the number of seconds remaining on thei
            watchdog timer. If the watchdog is not armed, returns -1.
        """
```

### There are 2 types of HW CPLD watchdog implementations on Mellanox [1] ###

#### Type 1 (Spectrum based devices) ####

- actual HW timeout can be defined as power of 2 msec;
e.g. timeout 20 sec will be rounded up to 32768 msec.; maximum timeout period is 32 sec (32768 msec.);
- get time-left isn't supported


#### Type 2 (Spectrum-2 based devices) ####

- actual HW timeout is defined in sec. and it's a same as user defined timeout; maximum timeout is 255 sec
- get time-left is supported

### Watchdog Plugin implementation ###

Watchdog can be configured using <b>ioctl</b>

| ioctl command | Param | Comment |
|---------------|-------|---------|
|WDIOC_KEEPALIVE| -| Ping watchdog
|WDIOC_SETTIMEOUT| timeout| Set timeout, return is actual timeout
|WDIOC_GETTIMEOUT| timeout| Get timeout
|WDIOC_GETTIMELEFT| timeleft| Get timeleft
|WDIOC_SETOPTIONS|WDIOS_DISABLECARD/WDIOS_ENABLECARD| Turn off/on watchdog


Common logic will be implemented in WatchdogImplBase class. WatchdogType1, WatchdogType2 inherit from WatchdogImplBase.

Because of Watchdog Type 1 does not support "get time-left" operation it should overwrite arm(), get_remaining_time() methods

Based on which type is availbale in the system Chassis class inits WatchdogType1 or WatchdogType2 object

#### Class diagram ####
![](https://github.com/stepanblyschak/SONiC/blob/wd/doc/pmon/wd_class_diagram.png)

#### Flow diagram ####
- WD is armed

![](https://github.com/stepanblyschak/SONiC/blob/wd/doc/pmon/wd_arm1.png)

- WD is not armed

![](https://github.com/stepanblyschak/SONiC/blob/wd/doc/pmon/wd_arm2.png)

WatchdogType1 should overwrite arm() and get_remaining_time()

arm() will save timestamp; get_remaining_time() will return (current_timeout - (current_timestamp - arm_timestamp))

### References ###
1. https://github.com/Mellanox/hw-mgmt/blob/V.2.0.0120/recipes-kernel/linux/linux-4.9/0017-watchdog-mlx-wdt-introduce-watchdog-driver-for-Mella.patch#L19
2. https://www.kernel.org/doc/Documentation/watchdog/watchdog-api.txt
