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

The more time mlxsw_minimal driver takes to initialize the more chance for race condition to happen in sequential swss restarts. With current timings on SPC1 - mlxsw_minimal takes ~2 sec to read and finish SFP device creation. For SPC2 without I2C bus overclocking it is around ~15 sec. In case it takes time which is long enough in order to execute another ```systemctl restart swss``` ASIC reset might happen while mlxsw_minimal accesses FW.
Even though ```sxdkernel stop``` will send REMOVE udev event - it doesn't mean hw-mgmt will cancel mlxsw_minimal driver in time before reset happens.

The race condition can be triggered by two sequential sx_core ```pcidrv_restart```'s:

```bash
root@sonic:~$ echo pcidrv_restart > /proc/mlx_sx/sx_core ; echo pcidrv_restart > /proc/mlx_sx/sx_core
root@sonic:~$ dmesg | grep 'reset\|mlxsw_minimal\|on sxcore'
[  978.689525] on sxcore event => chipdown starting
[  978.744434] i2c i2c-2: delete_device: Deleting device mlxsw_minimal at 0x48
[  979.518184] on sxcore event => chipdown finished
[  979.603042] sx_core 0000:01:00.0: reset trigger is already set
[  979.603043] Performing chip reset in this phase
[  979.603043] sx_core: performing SW reset
[  979.707497] sx_core 0000:01:00.0: SX_CMD_ACCESS_REG. Got FW status 0x26 after SW reset
[  981.738972] reset: system_enabled change to [true], time: 0[ms]
[  981.786066] on sxcore event => chipup starting
[  981.852707] mlxsw_minimal 2-0048: mlxsw_minimal mb size=100 off=0x00085058 out mb size=100 off=0x00085158
[  982.015140] mlxsw_minimal 2-0048: The firmware version 13.2000.2602
[  983.793388] sx_core 0000:01:00.0: reset trigger is already set
[  983.793390] Performing chip reset in this phase
[  983.793391] sx_core: performing SW reset
[  983.855897] mlxsw_minimal 2-0048: Reg cmd access status failed (status=7f(*UNKNOWN*))
[  983.898364] sx_core 0000:01:00.0: SX_CMD_ACCESS_REG. Got FW status 0x26 after SW reset
[  983.949838] mlxsw_minimal 2-0048: Reg cmd access failed (reg_id=900a(mtmp),type=query)
[  984.686132] mlxsw_minimal 2-0048: Fail to register core bus
[  984.753047] mlxsw_minimal: probe of 2-0048 failed with error -5
[  984.753056] i2c i2c-2: new_device: Instantiated device mlxsw_minimal at 0x48
[  984.758705] on sxcore event => chipup finished
[  984.763013] i2c i2c-2: delete_device: Deleting device mlxsw_minimal at 0x48
[  984.798783] on sxcore event => chipdown starting
[  984.856735] on sxcore event => chipdown finished
[  985.929763] reset: system_enabled change to [true], time: 0[ms]
[  985.979618] on sxcore event => chipup starting
[  986.041110] mlxsw_minimal 2-0048: mlxsw_minimal mb size=100 off=0x00085058 out mb size=100 off=0x00085158
[  986.204994] mlxsw_minimal 2-0048: The firmware version 13.2000.2602
[  989.915397] mlxsw_minimal 2-0048: Firmware revision: 13.2000.2602
[  989.915421] i2c i2c-2: new_device: Instantiated device mlxsw_minimal at 0x48
[  989.921089] on sxcore event => chipup finished
[  994.252576] mlxsw_minimal 2-0048 sfp1: renamed from eth1
...
```


### Fast Boot:

![Fast Boot flow](/doc/mlnx_only_doc/hw-mgmt-sxcore-fast-boot.svg)

fast boot is special because it will receive two sxcore ADD udev events.
One is a fake one from sxdkernel start and one about real ASIC reset

1) hw-mgmt chiupdis -> tell hw-mgmt to ignore exactly one sxcore ADD udev event
2) sxdkernel start -> does not perform ASIC reset but generates sxcore ADD udev event -> hw-mgmt ignores
3) SAI executes CTRL_CMD_RESET which resets ASIC -> sxcore ADD udev event is generated -> hw-mgmt starts mlxsw_mininal driver.

Expected dmesg logs with added debug prints for ADD/REMOVE udev events action handler:

```
admin@sonic:~$ sudo dmesg | grep 'reset\|mlxsw_minimal\|on sxcore'
[   34.002066] sx_core 0000:03:00.0: reset trigger is already set
[   34.002067] Did not perform chip reset in this phase
[   34.135067] on sxcore event => chipup starting
[   34.515143] on sxcore event => chipup finished
[   44.100412] on sxcore event => chipdown starting
[   44.575615] on sxcore event => chipdown finished
[   45.330315] sx_core 0000:03:00.0: reset trigger is already set
[   45.330316] Performing chip reset in this phase
[   45.330316] sx_core: performing SW reset
[   45.436503] sx_core 0000:03:00.0: SX_CMD_ACCESS_REG. Got FW status 0x26 after SW reset
[   47.466008] reset: system_enabled change to [true], time: 0[ms]
[   47.591674] on sxcore event => chipup starting
[   48.015920] mlxsw_minimal 2-0048: mlxsw_minimal mb size=100 off=0x00085058 out mb size=100 off=0x00085158
[   48.189249] mlxsw_minimal 2-0048: The firmware version 13.2000.2602
[   49.905403] mlxsw_minimal 2-0048: Firmware revision: 13.2000.2602
[   49.905427] i2c i2c-2: new_device: Instantiated device mlxsw_minimal at 0x48
[   50.027211] on sxcore event => chipup finished
[   54.252515] mlxsw_minimal 2-0048 sfp1: renamed from eth2
...
```

### Warm boot:

![Warm Boot flow](/doc/mlnx_only_doc/hw-mgmt-sxcore-warm-boot.svg)

1) sxdkernel start -> does not perform ASIC reset but generates sxcore ADD udev event -> hw-mgmt starts mlxsw_minimal driver
2) SAI skips CTRL_CMD_RESET.

Expected dmesg logs with added debug prints for ADD/REMOVE udev events action handler:

```
admin@sonic:~$ sudo dmesg | grep 'reset\|mlxsw_minimal\|on sxcore'
[   32.277141] sx_core 0000:03:00.0: reset trigger is already set
[   32.277142] Did not perform chip reset in this phase
[   32.408965] on sxcore event => chipup starting
[   32.950836] mlxsw_minimal 2-0048: mlxsw_minimal mb size=100 off=0x00085058 out mb size=100 off=0x00085158
[   33.170651] mlxsw_minimal 2-0048: The firmware version 13.2000.2602
[   34.777948] mlxsw_minimal 2-0048: Firmware revision: 13.2000.2602
[   34.777971] i2c i2c-2: new_device: Instantiated device mlxsw_minimal at 0x48
[   34.875753] on sxcore event => chipup finished
[   40.857270] mlxsw_minimal 2-0048 sfp2: renamed from eth3
...
```

### Reload:

#### Shutdown

![Syncd shutdown flow](/doc/mlnx_only_doc/hw-mgmt-sxcore-syncd-stop.svg)

#### Start - same as in cold boot mode

![Syncd startup flow](/doc/mlnx_only_doc/hw-mgmt-sxcore-cold-boot.svg)

1) sxdkernel stop -> generates sxcore REMOVE udev event -> hw-mgmt removes mlxsw_minimal driver
2) same as cold boot

Expected dmesg logs with added debug prints for ADD/REMOVE udev events action handler:

```
admin@sonic:~$ sudo config reload -y
...
admin@sonic:~$ sudo dmesg | grep 'reset\|mlxsw_minimal\|on sxcore'
[  445.597806] on sxcore event => chipdown starting
[  445.656260] i2c i2c-2: delete_device: Deleting device mlxsw_minimal at 0x48
[  446.411368] on sxcore event => chipdown finished
[  485.383556] sx_core 0000:01:00.0: reset trigger is already set
[  485.383557] Performing chip reset in this phase
[  485.383558] sx_core: performing SW reset
[  485.488974] sx_core 0000:01:00.0: SX_CMD_ACCESS_REG. Got FW status 0x26 after SW reset
[  487.496922] reset: system_enabled change to [true], time: 0[ms]
[  487.573794] on sxcore event => chipup starting
[  487.926786] mlxsw_minimal 2-0048: mlxsw_minimal mb size=100 off=0x00085058 out mb size=100 off=0x00085158
[  488.223478] mlxsw_minimal 2-0048: The firmware version 13.2000.2602
[  493.362015] mlxsw_minimal 2-0048: Firmware revision: 13.2000.2602
[  493.362040] i2c i2c-2: new_device: Instantiated device mlxsw_minimal at 0x48
[  493.449950] on sxcore event => chipup finished
[  495.602928] mlxsw_minimal 2-0048 sfp3: renamed from eth3
...

```


# Mannual testing

Verification
- mlxsw_minimal is not active while ASIC reset is performed
- mlxsw_minimal is activated only once
- no errors in dmesg/syslog related to sx_core/mlxsw_minimal
- the number of sfp netdevs is the number of modules in system regardless of port breakout configuration.

1) cold boot
2) fast boot
3) warm boot
4) config reload

# Automated testing

1) Run T0/T1-LAG regression
2) Run platform tests


