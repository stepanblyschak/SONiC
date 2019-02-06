# Watchdog plugin on Mellanox platforms design #

### Rev 0.1 ###

### Revision ###

 | Rev |     Date    |       Author       | Change Description                |
 |:---:|:-----------:|:------------------:|-----------------------------------|
 | 0.1 |  01/21/2019           |      Stepan Blyshchak      | Initial version        |
 
### Requirements ###

https://github.com/Azure/sonic-platform-common/blob/master/sonic_platform_base/watchdog_base.py
```python
class WatchdogBase:
    """
    Abstract base class for interfacing with a hardware watchdog module
    """

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
        raise NotImplementedError

    def disarm(self):
        """
        Disarm the hardware watchdog

        Returns:
            A boolean, True if watchdog is disarmed successfully, False if not
        """
        raise NotImplementedError

    def is_armed(self):
        """
        Retrieves the armed state of the hardware watchdog.

        Returns:
            A boolean, True if watchdog is armed, False if not
        """
        raise NotImplementedError

    def get_remaining_time(self):
        """
        If the watchdog is armed, retrieve the number of seconds remaining on
        the watchdog timer

        Returns:
            An integer specifying the number of seconds remaining on thei
            watchdog timer. If the watchdog is not armed, returns -1.
        """
        raise NotImplementedError
```

### There are 2 types of HW CPLD watchdog implementations on Mellanox [1] ###

#### Type 1 ####

- actual HW timeout can be defined as power of 2 msec;
e.g. timeout 20 sec will be rounded up to 32768 msec.; maximum timeout period is 32 sec (32768 msec.);
- get time-left isn't supported


#### Type 2 ####

- actual HW timeout is defined in sec. and it's a same as user defined timeout; maximum timeout is 255 sec
- get time-left is supported

There can be main and auxiliary watchdogs. Since API does not support multiple watchdogs we will focus on main watchdog.

### Current assumptions ###

- Watchdog daemon will arm the watchdog with <timeout> using ```arm(timeout)``` once on start
- Call ```arm()``` passing the same value <timeout> in a loop every <interval> seconds
- Watchdog daemon will not ignore ```arm()``` return value
- If watchdog daemon needs to change timeout it can call ```arm(new_timeout)```

### Watchdog Plugin implementation ###

Plugin will use standard linux API to interact with watchdog using <b>ioctl</b> commands:

| ioctl command     | Param                            | Comment                               | Supported by  |
|-------------------|----------------------------------|---------------------------------------|---------------|
|WDIOC_KEEPALIVE    | -                                | Ping watchdog                         | Type 1 & 2    |
|WDIOC_SETTIMEOUT   | timeout                          | Set timeout, return is actual timeout | Type 1 & 2    |
|WDIOC_GETTIMEOUT   | timeout                          | Get timeout                           | Type 1 & 2    |
|WDIOC_GETTIMELEFT  | timeleft                         | Get timeleft                          | Type 2        |
|WDIOC_SETOPTIONS   |WDIOS_DISABLECARD/WDIOS_ENABLECARD| Turn off/on watchdog                  | Type 1 & 2    |

<b>NOTE</b>: Since WDIOC_GETTIMELEFT is not supported on Type 1 and API does not assume it cannot be supported, get_remaining_time() for Type 1 will return time calculated using timestamps:
<p>

```time_left = (current_timeout - (current_timestamp - arm_timestamp))```

#### Class relations ####

Common logic for both will be implemented in ```WatchdogImplBase``` class that implements ```WatchdogBase``` API.
<p>

```WatchdogType1```, ```WatchdogType2``` inherit from ```WatchdogImplBase```.

Because of ```WatchdogType1``` does not support "get_remaining_time" operation it should overwrite arm(), get_remaining_time():
 - arm(): call arm() from base class and save the timestamp into object member variable
 - get_remaining_time(): Calculate time left using formula above

```Chassis``` object holds a reference to ```WatchdogBase```. On start it decides whether to create ```WatchdogType1``` or ```WatchdogType2```:

```Chassis``` object will list available watchdog devices in ```/dev/watchdog*``` and find the one with identity "mlnx-wdt-main" which is main Mellanox watchdog;
Be checking ```/sys/class/watchdog/watchdog{wd_index}/timeleft``` existance ```Chassis``` can distinguish between Type 1 and Type 2 and create ```WatchdogType1``` or ```WatchdogType2``` object.
If "mlnx-wd-main" is not available or error happened it set ```_watchdog``` variable to ```None```

The watchdog daemon will call ```get_watchdog()``` to get watchdog object.

#### Arm flow diagram ####
- WD is armed

![](https://github.com/stepanblyschak/SONiC/blob/wd/doc/pmon/wd_arm.png)

- On error
  - set previous watchdog armed state and timeout

### Open questions ###

1. Could it be better to have seperate API to arm and ping watchdog?

### References ###
0. https://github.com/Azure/sonic-platform-common/blob/master/sonic_platform_base/watchdog_base.py
1. https://github.com/Mellanox/hw-mgmt/blob/V.2.0.0120/recipes-kernel/linux/linux-4.9/0017-watchdog-mlx-wdt-introduce-watchdog-driver-for-Mella.patch#L19
2. https://www.kernel.org/doc/Documentation/watchdog/watchdog-api.txt
