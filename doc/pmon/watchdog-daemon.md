# Watchdog daemon design #

### Rev 0.1 ###

### Revision ###

 | Rev |     Date    |       Author       | Change Description                |
 |:---:|:-----------:|:------------------:|-----------------------------------|
 | 0.1 |  01/21/2019           |      Stepan Blyshchak      | Initial version        |
 
### Requirements ###
-	Watchdog should not be armed automatically. It is the SW daemon responsibility to arm it. It will be done during init as well as every <b>X</b> seconds based on the timeout defined

- It is the SW responsibility to re-am the watchdog. In the case it is not armed a restart should be performed by the HW watchdog and reboot reason should be set accordingly

- Upon user request the daemon can disarm the watchdog 

- Upon user request the daemon can set a new timeout. Once the timeout is set, the HW watchdog should be armed with the new value. Then every x time seconds based on the new timeout defined

- Upon user request the daemon should return the time left till the next arm. 
In this case I am not sure if it is ok to have the value returned by a SW timer or from the HW watchdog. Need to clarify.

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
```

### DB ###
#### State DB ####
#### Watchdog table ####

	; Defines information for a watchdog
	key                     = WATCHDOG_INFO                  ; information for the HW watchdog
	; field                 = value
	arm_status              = STRING                         ; Whether the HW WD is armed or not
	timeout                 = STRING                         ; Configured HW WD timeout
	expire_time             = STRING                         ; Expire time - how many seconds before HW WD expires
 
#### Config DB ####
#### Watchdog daemon table ####

	; Defines configuration for a watchdog daemon
	key                     = WATCHDOG_INFO                  ; information for the HW watchdog
	; field                 = value
	arm_status              = STRING                         ; Whether the HW WD is armed or not
	timeout                 = STRING                         ; Configured HW WD timeout
	expire_time             = STRING                         ; Expire time - how many seconds before HW WD expires

	
