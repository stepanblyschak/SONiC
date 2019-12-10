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

Expected dmesg logs:

```
admin@sonic:~$ sudo dmesg | grep 'reset\|mlxsw_minimal'
[   30.802720] sx_core 0000:01:00.0: reset trigger is already set
[   30.802721] Performing chip reset in this phase
[   30.802722] sx_core: performing SW reset
[   30.910571] sx_core 0000:01:00.0: SX_CMD_ACCESS_REG. Got FW status 0x26 after SW reset
[   32.930473] reset: system_enabled change to [true], time: 0[ms]
[   33.182523] mlxsw_minimal 2-0048: mlxsw_minimal mb size=100 off=0x00085058 out mb size=100 off=0x00085158
[   33.394587] mlxsw_minimal 2-0048: The firmware version 13.2000.2602
[   39.066115] mlxsw_minimal 2-0048: Firmware revision: 13.2000.2602
[   39.066899] i2c i2c-2: new_device: Instantiated device mlxsw_minimal at 0x48
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


