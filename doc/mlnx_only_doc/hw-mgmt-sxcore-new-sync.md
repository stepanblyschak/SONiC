# ASIC reset and access syncronization flow update in SONiC

## Hardware management suite and sx_core driver interaction

sx_core driver now implements generation of ADD/REMOVE udev event on sx_core_init_one_pci/sx_core_remove_one_pci.<br>
sx_core_init_one_pci is called when ASIC reset is done.<br>
hw-mgmt package relies on those events to call chipup/chipdown internally without OS interaction.

## Hardware management suite and MGPIR register

Regardless of the state of PMLP register, mlxsw_minimal driver now uses MGPIR register which has static information about the number of modules.
No synchronizations based on PortInitDone is required here.

## New flows

### Cold Boot
![Cold Boot flow](/doc/mlnx_only_doc/hw-mgmt-sxcore-cold-boot.svg)

1) sxdkernel start -> performs ASIC reset and generates sxcore ADD udev event -> hw-mgmt starts mlxsw_minimal driver
2) SAI executes CTRL_CMD_RESET which has no effect on ASIC reset.

Expected dmesg logs with added debug prints for ADD/REMOVE udev events action handler:

```
admin@sonic:~$ sudo dmesg | grep 'reset\|mlxsw_minimal\|on sxcore'
[   30.191012] sx_core 0000:01:00.0: reset trigger is already set
[   30.191013] Performing chip reset in this phase
[   30.191014] sx_core: performing SW reset
[   30.298659] sx_core 0000:01:00.0: SX_CMD_ACCESS_REG. Got FW status 0x26 after SW reset
[   32.322557] reset: system_enabled change to [true], time: 0[ms]
[   32.407535] on sxcore event => chipup starting
[   32.603241] mlxsw_minimal 2-0048: mlxsw_minimal mb size=100 off=0x00085058 out mb size=100 off=0x00085158
[   32.824790] mlxsw_minimal 2-0048: The firmware version 13.2000.2602
[   38.898654] mlxsw_minimal 2-0048: Firmware revision: 13.2000.2602
[   38.899002] i2c i2c-2: new_device: Instantiated device mlxsw_minimal at 0x48
[   39.174456] on sxcore event => chipup finished
[  133.508345] mlxsw_minimal 2-0048 sfp6: renamed from eth6
...
```

**NOTE**:
The nature of udev event is asynchronous - mlxsw_mininal driver initialization is done seperately from syncd start script process.
An ADD event is used only to start mlxsw_minimal driver, but it doesn't mean sxcore will wait for mlxsw_minimal initialization to finish. Because of that there is a posibility for a race condition when ```systemctl restart swss``` happends while mlxsw_minimal driver initializes.

With current timings - mlxsw_minimal takes ~2 sec to read and finish SFP device creation. In case it takes time which is long enough in order to execute another ```systemctl restart swss``` ASIC reset might happen while mlxsw_minimal accesses FW.
Even though ```sxdkernel stop``` will send REMOVE udev event - it doesn't mean hw-mgmt will cancel mlxsw_minimal driver in time before reset happens.

The race condition can be triggered by two sequential sx_core ```pcidrv_restart```'s:

```bash
root@sonic:~# echo pcidrv_restart > /proc/mlx_sx/sx_core ; echo pcidrv_restart > /proc/mlx_sx/sx_core
```


### Fast Boot:

![Fast Boot flow](/doc/mlnx_only_doc/hw-mgmt-sxcore-fast-boot.svg)

fast boot is special because it will receive two sxcore ADD udev events.
One is a fake one from sxdkernel start and one about real ASIC reset

1) hw-mgmt chiupdis -> tell hw-mgmt to ignore exactly one sxcore ADD udev event
2) sxdkernel start -> does not perform ASIC reset but generates sxcore ADD udev event -> hw-mgmt ignores
3) SAI executes CTRL_CMD_RESET which resets ASIC -> sxcore ADD udev event is generated -> hw-mgmt starts mlxsw_mininal driver.

### Warm boot:

![Warm Boot flow](/doc/mlnx_only_doc/hw-mgmt-sxcore-warm-boot.svg)

1) sxdkernel start -> does not perform ASIC reset but generates sxcore ADD udev event -> hw-mgmt starts mlxsw_minimal driver
2) SAI skips CTRL_CMD_RESET.

### reload flow:

1) sxdkernel stop -> generates sxcore REMOVE udev event -> hw-mgmt removes mlxsw_minimal driver
2) same as cold boot



# Mannual testing

Verification: mlxsw_minimal is not active while ASIC reset is performed and is activated only once, no errors in dmesg/syslog,
the number of sfp netdevs is the number of modules in system regardless of port breakout configuration.

1) cold boot
2) fast boot
3) warm boot
4) config reload

# Automated testing

1) Run T0/T1-LAG regression
2) Run platform tests


