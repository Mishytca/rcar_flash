# rcar_flash

## About

`rcar_flash` is a simple command line tool to automate writing IPLs
(firmware) to Renesas RCAR-based boards. It provides configurations
for many different boards and it is relatively easy to add support for
a new board. It is capable of controlling CPLD and uploading serial
download module (Flash Writer) to a board, which provides hassle-free
way to update bootloaders on your setup.

It works by interacting with MiniMonitor/FlashWriter software running
on the board via serial console. If you are tired of typing `xls2`
commands into MiniMonitor again and again, this tool is for you.

- [Quickstart](#quickstart)
  - [Initial setup](#initial-setup)
  - [Flashing your board](#flashing-your-board)
    - [Choose your config](#choose-your-config)
    - [Fully automated flashing with CPLD support](#fully-automated-flashing-with-cpld-support)
    - [Flashing the board that is in MiniMonitor mode](#flashing-the-board-that-is-in-minimonitor-mode)
    - [Flashing the board that is in serial download mode](#flashing-the-board-that-is-in-serial-download-mode)
    - [Flashing only some bootloaders](#flashing-only-some-bootloaders)
- [Reference manual](#reference-manual)
  - [Select configuration file](#select-configuration-file)
  - [`list-boards` sub-command](#list-boards-sub-command)
  - [`list-loaders` sub-command](#list-loaders-sub-command)
  - [`flash` sub-command](#flash-sub-command)
  - [YAML file "schema"](#yaml-file-schema)
    - [`flash_target`](#flash_target)
    - [`cpld_profiles`](#cpld_profiles)
    - [`board`](#board)
  - [Work with binary files](#work-with-binary-files)
  - [Run flash_writer only](#run-flash_writer-only)
- [Board-specific instructions](#board-specific-instructions)
  - [Salvator-X(S)](#salvator-xs)
  - [Whitehawk V4H](#whitehawk-v4h)
- [Examples](#examples)

## Quickstart

### Initial setup

You need to perform a number of steps to make `rcar_flash` work on
your machine.

Option 1 (preferred installation method):
Install the `rcar_flash` tool as a package using `pipx`:

```
pipx install git+https://github.com/xen-troops/rcar_flash.git
```

This will allow you to run the `rcar_flash` tool directly from the command line.
For example:

```
rcar_flash -h
```

Option 2.
You can to clone the repository somewhere on your PC:

```
# git clone https://github.com/xen-troops/rcar_flash.git
```

And install two packages: `pyserial` and `pyftdi`. This depends on
your Linux distribution, but for Ubuntu the following command should
install pyserial:

```
# sudo apt install python3-serial
```

You need to have the proper rights for access to serial devices
like `/dev/ttyUSB0`. As a general recommendation, a user has to
be added to the `dialout` group.

As for `pyftdi` it is a little more trickier, so please check the
pyftdi manual for the installation and setting proper access rights:
<https://eblot.github.io/pyftdi/installation.html>. `pyftdi` required
only if you are planning to use CPLD control functionality, but we are
strongly suggest to install it.

### Flashing your board

#### Choose your config

Use

```
# ./rcar_flash.py list-boards
```

to see list of all currently supported boards. Choose the most
appropriate for you. If you are working with Starter Kit (ulcb) or
Spider (s4) boards, you are in luck, because you can use CPLD to
automatically switch the board to a serial download mode.

#### Fully automated flashing with CPLD support

The next command will try to flash all known loaders located at
`<path-to-loader-files>` for board `<board-name>`.

```
# ./rcar_flash.py flash -b <board-name> -c -f -p <path-to-loader-files> all
```

It will try to determine your board USB interface automatically, but
it might fail if there are multiple compatible USB devices connected
to your PC. In this case you need to pass serial number for correct USB manually:

```
# ./rcar_flash.py flash -b <board-name> -c <serial-no> -f -p <path-to-loader-files> all
```


#### Flashing the board that is in MiniMonitor mode

If `rcar_flash` does not support CPLD control for your board or you
don't want to use it, you can place board into MiniMonitor mode (not
to be confused with serial download mode) manually (check manual for
your board, for example
<https://elinux.org/R-Car/Boards/H3SK#Flashing_firmware>) and then
flash all loaders to your board:

```
# ./rcar_flash.py flash -b <board-name> -s /dev/ttyUSB0 -p <path-to-loader-files> all
```

Replace `/dev/ttyUSB0` with the correct USB serial port name. In this
mode `rcar_flash` will start writing bootloaders right away.


#### Flashing the board that is in serial download mode

If `rcar_flash` does not support CPLD control for your board or
you don't want to use it, you can place board in serial download mode
(not to be confused with MiniMonitor mode) manually (check manual for
your board, for example
<https://elinux.org/R-Car/Boards/H3SK#Flashing_firmware>) and then
flash all loaders to your board:

```
# ./rcar_flash.py flash -b <board-name> -s /dev/ttyUSB0 -f -p <path-to-loader-files> all
```

Replace `/dev/ttyUSB0` with the correct USB serial port name. In this
mode, `rcar_flash` will send Flash Writer prior to writing any
bootloaders.

#### Flashing only some bootloaders

Replace `all` keyword from previous sections with list of loaders you
wish to flash. For example, instead of `all` write `bl31 u-boot tee`
to flash only BL31, U-Boot and OP-TEE. You can get list of loaders
available for your board with the following command:

```
# ./rcar_flash list-loaders -b <board-name>
```

## Reference manual

### Select configuration file

By default `rcar_flash` reads all required data from shipped `rcar_flash.yaml`
file. But you may provide your own file, using the option `--conf`:

```
# ./rcar_flash.py --conf my_config.yaml
```

### `list-boards` sub-command

This command is used to list all supported boards. It has no
additional parameters. Example:


    # ./rcar_flash list-boards
    Board Name          Default flash loader file
    e3_2x512            AArch64_Gen3_E3_Scif_MiniMon_develop_Ebisu_V0.02.mot
    e3_4x512            AArch64_Gen3_E3-4D_Scif_MiniMon_V5.03A.mot
    h3_2x2              AArch64_Gen3_H3_M3_Scif_MiniMon_V3.03.mot
    h3_4x1              AArch64_Gen3_H3_M3_Scif_MiniMon_V3.03.mot
    h3_4x2              AArch64_Gen3_H3_M3_Scif_MiniMon_V3.03.mot
    h3ulcb              AArch32_Flash_writer_SCIF_DUMMY_CERT_E6300400_ULCB.mot
    h3ulcb_4x2          AArch32_Flash_writer_SCIF_DUMMY_CERT_E6300400_ULCB.mot
    m3                  AArch64_Gen3_H3_M3_Scif_MiniMon_V5.11.mot
    m3_2x4              AArch64_Gen3_H3_M3_Scif_MiniMon_V5.11.mot
    m3n                 AArch64_Gen3_Scif_MiniMon_Develop_M3N_V0.03.mot
    m3ulcb              AArch32_Flash_writer_SCIF_DUMMY_CERT_E6300400_ULCB.mot
    s4                  ICUMX_Flash_writer_SCIF_DUMMY_CERT_EB203000_S4.mot
    v4h                 ICUMX_Flash_writer_SCIF_DUMMY_CERT_EB203000_V4H.mot

### `list-loaders` sub-command

This command is used to list all known bootloaders for a given
board. It has one mandatory argument:

- `-b/--board BOARD` - provide name of board, for which you
  want to list bootloaders.

Example:

    #  ./rcar_flash.py list-loaders -b s4
    [INFO] Reading config for board s4
    Loader                       Flash address       Default file                             Flash target
    ------------------------------------------------------------------------------------------------------
    bl31                         0x7000              bl31-spider.srec                         s4_emmc
    tee                          0x7400              tee-spider.srec                          s4_emmc
    u-boot                       0x7c00              u-boot-elf.srec                          s4_emmc
    bootparam                    0x0                 bootparam_sa0.srec                       s4_qspi
    icumx_loader                 0x40000             icumx_loader.srec                        s4_qspi
    cert_header                  0x240000            cert_header_sa9.srec                     s4_qspi
    dummy_fw                     0x280000            dummy_fw.srec                            s4_qspi
    dummy_icumh_case1            0x380000            dummy_icumh_case1.srec                   s4_qspi
    ca55_loader                  0x480000            ca55_loader.srec                         s4_qspi
    cr52                         0x500000            App_CDD_ICCOM_S4_Sample_CR52.srec        s4_qspi
    g4mh                         0x900000            App_CDD_ICCOM_S4_Sample_G4MH.srec        s4_qspi


### `flash` sub-command

This is the sub-command that you will use most often. It tries to
flash selected bootloaders onto a selected board.

Required parameters:

- `-b/--board BOARD` - name of board you wish to flash

- `loaders [loaders ...]` - list of loaders you want to flash. It can
  be either `all` to flash all loaders, or list of (pairs):
  `bl2:bl2.srec bl31:bl31.srec u-boot`. You can provide either just
  bootloader name (like `bl2`) or bootloader name and corresponding
  file name (like `bl2:bl2.srec`). If filename is not provided,
  `rcar_flash` will use default filename for that bootloader. You can
  check default names with `list-loaders` sub-command.

Optional parameters:

- `-h` - just show embedded help message.

- `-c/--cpld [serial_no]` - try to use CPLD to switch board into
  serial download mode. Optional `serial_no` can be used to choose
  correct USB device. Implies `-f`.

- `-f/--flash-writer [flashwriter.mot]` - send flash writer to the
  board before attempting to upload any bootloaders. Flash Writer is a
  special piece of code that is responsible of writing all other
  bootloaders into flash memory. It can be found at
  <https://github.com/renesas-rcar/flash_writer>. Optional
  `[flashwriter.mot]` parameter allows you to override default flash
  writer name. Use this option only if your board is already in serial
  download mode and awaits for flash-writer to be sent, or if you are
  using `-c` parameter, in this case `rcar_flash` will put the board
  into serial download mode for you`.

- `-p/--path PATH` - path to bootloader files. By default `rcar_flash`
  tries to find bootloader files in the current working directory. You
  can provide another path (for example, `deploy` dir of your Yocto
  build).

- `-s/--serial SERIAL` - path to serial device (like
  `/dev/ttyUSB0`). By default `rcar_flash` will try to determine
  serial device automatically, but this is not always possible, you
  can override path.


If `-f` or `-c` parameters are not present, `rcar_flash` will assume
that board is in MiniMonitor/FlashWriter mode already and will try to
flash loaders right away.

### YAML file "schema"

`rcar_flash` reads all required data from shipped `rcar_flash.yaml`
file. This file tells `rcar_flash` how to communicate with different
board types and where to flash bootloaders. You might need to tailor
it for your specific needs. It has the following sections:

#### `flash_target`

This section describes "programs" used to flash different types of
boards. They allow users to tailor flash process for a specific
version of MiniMonitor/FlashWriter or to different targets, like
HyperFlash or eMMC. Those "programs" are simple lists of "wait-for" -
"send" commands. Example:

    s4_qspi:
      sequence:
        - wait_for: ">"
          send: const
          val: "xls2\r"
        - wait_for: "(1-3)>"
          send: const
          val: "1\r"
        - wait_for: "(Push Y key)"
          send: const
          val: "Y"
        - wait_for: "(Push Y key)"
          send: const
          val: "Y"
        - wait_for: "Please Input : H'"
          send: img_addr
        - wait_for: "Please Input : H'"
          send: flash_addr
        - wait_for: "(Motorola S-record)"
          send: file
        - wait_for: "(y/n)"
          send: const
          val: "y"

This example describes how to flash QSPI on s4/spider
board. `rcar_flash` will wait for ">", then send "xls2", then wait for
"(1-3)" message, choose option "1" and so on...

Allowed arguments for `send` are:

- `const` - send constant string defined in `val` field
- `img_addr` - send the image (load) address. This address is
  determined by parsing `.srec` header of the file.
- `file_size` - send the size of the binary file. It is obtained
   by the script automatically. Can be used for binary files only.
   See the "Work with binary files" section below.
- `flash_addr` - send the flash (store) address. This address is read
  from corresponding `ipl` entry (check out next parts of the
  documentation).
- `file` - send the bootloader file

Also in some specific cases, like flashing of big files, flash_writer
may be silent for 30 seconds or longer. In this case, you should
use the optional command `timeout`, providing a safe timeout in seconds.
Like this:

      - wait_for: "complete!"
        timeout: 40
        send: const
        val: "\r"

Specified timeout will be used only for mentioned `wait_for`.

#### `cpld_profiles`

This section defines how to communicate with CPLD and which registers
to update to switch board into different modes. In most cases you
don't need to alter those parameters, but if you really sure you need
to - you already will know what they mean :)

#### `board`

This section defines all known boards. Example for one board:

```
  m3ulcb:
    flash_writer: AArch32_Flash_writer_SCIF_DUMMY_CERT_E6300400_ULCB.mot
    sup_baud: 921600
    ipls:
      bootparam:
        file: bootparam_sa0.srec
        flash_addr: 0x0
        flash_target: gen3_hf
      bl2:
        file: bl2-m3ulcb.srec
        flash_addr: 0x40000
        flash_target: gen3_hf
      cert_header:
        file: cert_header_sa6.srec
        flash_addr: 0x180000
        flash_target: gen3_hf
      bl31:
        file: bl31-m3ulcb.srec
        flash_addr: 0x1C0000
        flash_target: gen3_hf
      tee:
        file: tee-m3ulcb.srec
        flash_addr: 0x200000
        flash_target: gen3_hf
      u-boot:
        file: u-boot-elf.srec
        flash_addr: 0x640000
        flash_target: gen3_hf
```

For each board there are multiple options possible:

- `flash_writer` - mandatory - defines default Flash Writer filename.

- `baud` - optional - defines baud rate used to communicate with the
  board. Default value is 115200.

- `sup_baud` - optional - defines "speed-up" baud rate. `rcar_flash`
  will try to use `sup` command to increase communication speed with
  the board. This might increase flashing speed considerably. There is
  no default value, as different boards allow different speeds.

- `cpld_profile` - optional - name of CPLD profile (check the previous
  section) which should be used to communicate with the board.

- `ipls` - mandatory - list of bootloaders that can be flashed to this
  board. Each entry should have the following options:
    - `file` - default file name for said bootloader
    - `flash_addr` - address in flash memory where to write this bootloader
    - `flash_target` - which "flash_target" to use while writing this
      bootloader. Flash targets are described in the one of the
      previous sections. This option is per-loader, not per-board
      because some boards (like `spider`) can have different targets for
      different loaders.

### Work with binary files

In case of need, you can flash not only .srec-files but also binaries.
For this, you should use a slightly modified `flash_target`:

- replace `xls2` with `xls3`

- replace `img_addr` with `file_size`

- replace `"please send ! ('.' & CR stop load)"` with `"please send ! (binary)"`

- provide corresponding .bin-files in the list of the loaders for the board

Please see below an example of the flash_target that allows flashing binary
files to HyperFlash on H3ULCB board:

```
flash_target:
  gen3_hf_bin:
    sequence:
      - wait_for: ">"
        send: const
        val: "xls3\r"
      - wait_for: "(1-3)>"
        send: const
        val: "3\r"
      - wait_for: "(Push Y key)"
        send: const
        val: "Y"
      - wait_for: "(Push Y key)"
        send: const
        val: "Y"
      - wait_for: "Please Input : H'"
        send: file_size
      - wait_for: "Please Input : H'"
        send: flash_addr
      - wait_for: "please send ! (binary)"
        send: file
      - wait_for: "(y/n)"
        send: const
        val: "y"
```

### Run flash_writer only

It is possible to load flash_writer only without flashing any loader.
This may be useful if someone wants to work with flash_writer commands.
In this case, you may specify `none` as the list of required loaders.
In the following example, `rcar_flash` will use CPLD to switch the board
to the download mode, load flash_writer specified for h3ulcb_4x2, and stop.
After that, you may open your serial port and work with flash_writer.

```
./rcar_flash.py flash -c -f -b h3ulcb_4x2 -s /dev/ttyUSB0 none
```

Pay attention to the speed of the serial console. If the board supports
SUP, then `sup_baud` will be used after loading the flash_writer.
You may identify this by the message like
```
[INFO] Using serial port /dev/ttyUSB0 with baudrate 921600
```

## Board-specific instructions

### Salvator-X(S)

Salvator X(S) boards have no CPLD so it is impossible to set boot mode
automatically. Thus, fully automated mode is impossible. User has two
choices: either boot into MiniMonitor, as described in Salvator HW manual
or Yocto Startup Guide, or boot into Serial Download mode. Booting
into serial download mode is easier, because fewer DIP switches need
to be changed.

To boot into download mode on your Salvator X(S) board do the following:

1. Turn off your board.
2. Locate SW10 DIP switches. They are positioned near the RGB
   connector, on the top side of the board.
3. Remember position of pins 5-8. You will need to set them back after
   flashing. In most cases they are set as following: ON, ON, OFF, ON,
   but your config may be different.
4. Configure serial download mode by setting pins 5-8 of SW10:

| Pin | Value |
|-----|-------|
| 5   | OFF   |
| 6   | OFF   |
| 7   | OFF   |
| 8   | OFF   |

5. Turn on your Salvator.
6. Upload your bootloaders:

```
./rcar_flash.py flash -f -b h3_4x2 -s /dev/ttyUSB0 all
```

7. Power off the board again.
8. Return pins 5-8 of SW10 back to original values from step 3.
9. Power on your board back.

### Whitehawk V4H

Whitehawk V4h has no CPLD so it is impossible to set boot mode
automatically. User has to boot the board into Serial Download mode
before flashing according to the following instruction:

1. Turn off your board.
2. Locate SW1 DIP switches. They are located on the small red board,
   in the row near the serial number, closer to the SOC.
3. Remember position of pins 5-8. You will need to set them back after
   flashing.
4. Configure serial download mode by setting pins 5-8 of SW1 to OFF position:

| Pin | Value |
|-----|-------|
| 5   | OFF   |
| 6   | OFF   |
| 7   | OFF   |
| 8   | OFF   |

5. Turn on your board.
6. Upload your bootloaders:

```
./rcar_flash.py flash -f -b v4h -s /dev/ttyUSB0 all
```

7. Power off the board again.
8. Return pins 5-8 of SW1 back to original values from step 3.
9. Power on your board back.

## Examples

List supported boards
```
./rcar_flash.py list-boards
```

List loaders for the specified board
```
./rcar_flash.py list-loaders -b <board>
```

Flash all loaders to the whitehawk (v4h) board that is already in flashing mode
```
./rcar_flash.py flash -b v4h -f -s /dev/ttyUSB0 -p <path> all
```

Flash only bl31 and u-boot to the whitehawk board
```
./rcar_flash.py flash -b v4h -f -s /dev/ttyUSB0 -p <path> bl31 u-boot
```

Flash only bl31 that has non-default name to the whitehawk
```
./rcar_flash.py flash -b v4h -f -s /dev/ttyUSB0 -p <path> bl31:my_bl31.srec u-boot
```

Flash all loaders to the spider (s4) board that supports CPLD to swith into the flashing mode
```
./rcar_flash.py flash -b s4 -c -f -s /dev/ttyUSB0 -p <path> all
```
