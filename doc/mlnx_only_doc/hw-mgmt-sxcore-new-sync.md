# ASIC reset and access syncronization flow update in SONiC

## Hardware management suite and sx_core driver interaction

sx_core driver now implements generation of ADD/REMOVE udev event on sx_core_init_one_pci/sx_core_remove_one_pci.<br>
sx_core_init_one_pci is called when ASIC reset is done.<br>
hw-mgmt package relies on those events to call chipup/chipdown internally without OS interaction.

# Hardware management suite and MGPIR register

Regardless of the state of PMLP register, mlxsw_minimal driver now uses MGPIR register which has static information about the number of modules.
No synchronizations based on PortInitDone is required here.

## New flows

### Cold Boot
![Cold Boot flow](/doc/mlnx_only_doc/hw-mgmt-sxcore-cold-boot.svg)

1) sxdkernel start -> performs ASIC reset and generates sxcore ADD udev event -> hw-mgmt starts mlxsw_minimal driver
2) SAI executes CTRL_CMD_RESET which has no effect on ASIC reset.

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


