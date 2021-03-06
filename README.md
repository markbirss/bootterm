# BootTerm

Bootterm is a simple, reliable and powerful terminal designed to ease connection to ephemeral serial ports as found on various SBCs, and typically USB-based ones.

## Main Features

- automatic port detection (uses the most recently registered one by default)
- enumeration of available ports with detected drivers and models
- wait for a specific, a new, or any port to appear (convenient with USB ports)
- support for non-standard baud rates (e.g. 74880 bauds for ESP8266)
- can send a Break sequence and toggling RTS/DTR for various reset sequences
- fixed/timed captures to files (may be enabled at run time)
- reliable with proper error handling
- single binary without annoying dependencies, builds out of the box
- supports stdin/stdout to inject/download data
- configurable escape character and visible character ranges

## Setup

Setting it up and installing it are fairly trivial. By default it will install in `/usr/local/bin` though this can be changed with the `PREFIX` variable, e.g. to install in ~/.local/bin instead:

```
$ git clone https://github.com/wtarreau/bootterm
$ cd bootterm
$ make install PREFIX=~/.local
$ (or "sudo make install" for a system-wide installation)
```

The program comes with no other dependency than a basic libc and produces a single binary (`bt`). It can easily be cross-compiled by setting `CROSS_COMPILE` or `CC`, though the makefile only adds unneeded abstraction and could simply be bypassed (please check it, it's self-explanatory). It was tested on several Linux distros and platforms (i386, x86_64, arm, aarch64).

## Using it

### Most common usage
By default, `bt` connects to the last registered serial port, which usually is the most recently connected USB adapter. A list of currently available ports is obtained by `bt -l`:

```
$ bt -l  
 port |  age (sec) | device     | driver           | model                
------+------------+------------+------------------+----------------------
    0 |     524847 | ttyAMA0    | uart-pl011       |                  
    1 |     524847 | ttyUSB0    | ftdi_sio         | TTL232R-3V3      
    2 |       1320 | ttyUSB1    | ch341-uart       |                  
 *  3 |        206 | ttyUSB2    | cp210x           | CP2102 USB to UART Bridge Controller 
```

In the example above, `ttyUSB2` will be used when no option is specified (which is indicated by the star in front of the port number). Otherwise, running `bt ttyUSB2`, `bt /dev/ttyS0`, or anything else will also do what is expected. The help is shown with `bt -h`.

Once connected, the default escape character is the same as telnet: Ctrl-] (Control and right-square-bracket). After the escape character, a single character is expected to issue an internal command. Quitting is achieved by pressing `q` (either upper or lower case) or `.` (dot) after the escape character. Sending the escape character itself is possible by issuing it again. A help page is available with `h` or `?`:

```
$ bt
No port specified, using ttyUSB0 (last registered). Use -l to list ports.
Trying port ttyUSB0... Connected to ttyUSB0 at 115200 bps.
Escape character is 'Ctrl-]'. Use escape followed by '?' for help.

BootTerm supports several single-character commands after the escape character:
  H h ?      display this help
  Q q .      quit
  P p        show port status
  D d        flip DTR pin
  R r        flip RTS pin
  F f        flip both DTR and RTS pins
  B b        send break
  C c        enable / disable capture
Enter the escape character again after this menu to use these commands.
```

### Waiting for a new serial port

Most often you don't want to know what name your serial port will have, you just want to connect to the one you're about to plug, as soon as it's available, in order to grasp most of the boot sequence. That's what `bt -n` is made for. It will check the list of current ports and will wait for a new one to be inserted. It even works if one port is unplugged and replugged. This is very convenient to avoid having to follow cables on a desk, or when connecting to a device that gets both the power and the console from the same USB connector.

Please note that the automatic port detection feature is only supported on Linux for now.

Example below booting a Breadbee board with an integrated CH340E adapter after `bt -n`:

```
$ bt -n
3 ports found, waiting for a new one...
 port |  age (sec) | device     | driver           | model                
------+------------+------------+------------------+----------------------
    3 |          0 | ttyUSB1    | ch341-uart       |                  

Trying port ttyUSB1... Connected to ttyUSB1 at 115200 bps.
Escape character is 'Ctrl-]'. Use escape followed by '?' for help.
!

U-Boot 2019.07-00068-g18b9e73630-dirty (May 30 2020 - 13:37:48 +0900)

DRAM:  64 MiB
```

One particularly appreciable case is when connecting to an emulated port, such as below on a NanoPI NEO2 running Armbian:
```
$ bt -n
2 ports found, waiting for a new one...
 port |  age (sec) | device     | driver           | model                
------+------------+------------+------------------+----------------------
    2 |          0 | ttyACM0    | cdc_acm          | CDC Abstract Control Model (ACM) 
```

When connecting to a new port, the baud rate is automatically set to 115200 bps. However it is not changed for already existing ports. This port selection mode can be automatically enabled by setting the `BT_SCAN_WAIT_NEW` environment variable to any value:

```
$ export BT_SCAN_WAIT_NEW=1
$ bt
3 ports found, waiting for a new one...
```

### Waiting for any port

A different approach consists in waiting for either a specific port or any port. By default, issuing `bt ttyUSB0` will fail if this terminal doesn't exist yet. But with `bt -a ttyUSB0`, BootTerm will wait for the device to appear.

With no device name specified, it will wait for any usable device and use the most recent one. If some ports are already on board and must never be used, they can be excluded using the environment variable `BT_SCAN_EXCLUDE_PORTS`:

```
$ export BT_SCAN_EXCLUDE_PORTS=ttyS0,ttyS1
$ bt -a
Waiting for one port to appear...
Port ttyUSB0 available, using it.
 port |  age (sec) | device     | driver           | model                
------+------------+------------+------------------+----------------------
    0 |          0 | ttyUSB0    | ch341-uart       |                  

Trying port ttyUSB0... Connected to ttyUSB0 at 115200 bps.
Escape character is 'Ctrl-]'. Use escape followed by '?' for help.
```

It is also possible to enable this port selection mode by default using `BT_SCAN_WAIT_ANY`:
```
$ export BT_SCAN_EXCLUDE_PORTS=ttyS0,ttyS1
$ export BT_SCAN_WAIT_ANY=1
$ bt
Waiting for one port to appear...
```

It is probably what most laptop users will want to do so as never to have to pass any argument and automatically connect to a USB serial port.

Alternately, it is possible to restrict the port enumeration to only a specific set by listing them in `BT_SCAN_RESTRICT_PORTS`. This can be more convenient when you know that your port is always called `ttyACM0` or any such thing for example.

### Changing the baud rate

By default, the port's baud rate is maintained for ports that were already present before `bt` started. It will be forced to 115200 bauds for newly discovered ports. However the speed may be always forced using the `BT_PORT_BAUD_RATE` environment variable, or by using `-b` followed by a speed. Example below with a NodeMCU module:

```
$ bt -b 74880
Port ttyUSB2 available, using it.
 port |  age (sec) | device     | driver           | model                
------+------------+------------+------------------+----------------------
    2 |     524342 | ttyUSB2    | cp210x           | CP2102 USB to UART Bridge Controller 

Trying port ttyUSB2... Connected to ttyUSB2 at 76800 bps.
Escape character is 'Ctrl-]'. Use escape followed by '?' for help.
```

It is interesting to note above that the hardware does not support 74880 bauds and selected its closest support speed (76800). This is a 2.5% error, it will not cause any trouble.

### Using bootterm to detect ports

Bootterm supports a "print" mode. In this mode it will simply print the device name without the leading `/dev/`. It can be convenient as an assistant to other flashing tools to wait for a port and print its name. By default the newly detected port is reported however, and it is wise to switch to quiet mode to avoid intermediary information:
```
$ bt -np
4 ports found, waiting for a new one...
ttyUSB2
```

```
$ bt -npq
ttyUSB2

```

Thus a script that needs to connect to the port as early as possible to reprogram a board could be doing something like this to wait for the device to show up:
```
PORT=$(bt -npq)
if [ -n "$PORT" ]; then
   flash -p "$PORT" -i "$IMAGE"
fi
```

### Issuing special sequences

The current port status (name, speed, pins) is reported when pressing `p` after the escape sequence. Pin names in lower cases are in the low state, those in upper case are in the high state. When pin toggling is required, it is wise to check the real pin's polarity, as most circuits invert it multiple times along the chain. The reported status here is the one seen by the serial port driver.

A break sequence can be used to trigger the SysRq feature in Linux, or to reboot some boards. The break happens by pressing `b` after the escape key. E.g. `Ctrl-] b`.

The DTR pin can be toggled by pressing `D` after the escape character, and the RTS pin can be toggled by pressing `R`. Both pins can be toggled together (for example to reset an ESP8266 in flashing mode) by pressing `F`.

### Capturing responses

There are two ways of capturing responses. If no terminal is needed, simply redirecting `bt`'s output to a file will work:

```
$ bt /dev/ttyS0 > server-panic.log
```

If a terminal is needed and a capture of the session is desired, there is a special capture function. It supports three modes, which are enabled either by `-c` on the command line, otherwise by environment variable `BT_CAPTURE_MODE`:

 - `none` : capture is disabled, this is the default
 - `fixed`: a file name is fixed on start of the capture and will not change
 - `timed`: the file name is re-evaluated each second and if it changes, the previous file is closed and a new one is opened.

In all cases, the files are only appended to and never truncated, so that bootterm will never destroy previous traces. If a fresh file is needed, just remove the file before starting bootterm.

The `fixed` mode usually is the one that users will want to capture a session when dumping a boot loader's configuration or the kernel's boot messages. The `timed` mode can be useful when connecting to a port waiting for rare or periodic events in order to get a dated file. The file name is defined by the format passed to `-f`, which defaults to the current date and time. It uses `strftime()` so please check this man page to get all format options supported on your system.

Thus when exploring a new device, one would likely use:
```
$ bt -c fixed
```

And when collecting event logs from a server with one file per day:
```
$ bt -c timed -f "server-%Y%m%d.log"
```

### Changing the escape character

The escape character may be specified either as a single character after -e or as an integer or hexadecimal value representing the ASCII code of this character.

Users coming from the venerable `screen` utility will probably want to set the escape character to Ctrl-A (unless of course they want to call screen from within these sessions, or are irritated by the confiscation of this common sequence):

```
$ bt -e1
```

Those coming from tmux would rather use Ctrl-B:
```
$ bt -e2
```

Here are a few other convenient and less common escape characters:

|Sequence | Code | Argument |
|---------|------|----------|
| Ctrl-@  |  0   | `-e0`    |
| Ctrl-A  |  1   | `-e1`    |
| Ctrl-Z  |  26  | `-e26`   |
| Esc     |  27  | `-e27`   |
| Ctrl-\  |  28  | `-e28`   |
| Ctrl-]  |  29  | `-e29`   |
| Ctrl-^  |  30  | `-e30`   |
| Ctrl-_  |  31  | `-e31`   |


### Masking problematic characters

Running at wrong speeds or connecting during a boot sequence often results in some garbage to be received on a terminal. And different terminal emulators handle these differently. For example, Xterm supports a very large variety of codes in the C1 range (0x80-0x9F) among which CSI (0x9B) which is an escape with the high bit set. The problem is that the prefix range is large and that many of them will result in reconfiguring it or hanging it until a sequence ends. This is often translated into unreadble fonts, the terminal definitely freezing, or arrow keys making the cursor move on the screen instead of being sent to the remote terminal. Other terminals have their own problems with 0x00, 0xFF or can be at risk with improper handling of ANSI codes prefixed with 0x1B.

In order to address this, bootterm can block dangerous characters and print them encoded instead. It will always block raw and UTF-8 encoded C1 codes (0x80-0x9F with or without the 0xC2 prefix), as these are always terminal configuration codes which should never be sent to the terminal emulator. In addition it's possible to restrict the range of printable characters using `-m` and `-M`.

## Motivation

The first motivation behind writing bootterm was that when working with single board computers (SBC), the serial port is the only interface available during most of the development or exploration of the board, and that sadly, the available tools are either pretty poorly designed, or incomplete as they were not initially made for this purpose. Some tools like `screen` use particularly inconvenient key mappings making it a real pain to use the line editing on some devices, others like `minicom` reconfigure the terminal disabling scrolling and making copy-paste a nightmare. Some like `cu` were not initially designed for this and nobody ever knows the right command-line options nor how to quit it. Then comes the myriad of Python 3-liners which do no error checking, resulting in spinning loops when a port is disconnected and the port being renumbered once reconnected, or conversely crashes and backtraces when facing binary character sequences that do not match UTF-8. Not to mention that many times they can't even install as they depend on various modules that are incompatible with those installed on the local machine.

The author has had the best experience with `kwboot`, a tool initially made for flashing some Marvell-based devices, which integrates a terminal that is started at the end of the transfer, and which is now part of U-Boot. A few fixes and changes were brought there (such as allowing the terminal to start without flashing), and it served the purpose reasonably well for a few years. But it doesn't support non-standard speeds that some chips require (ESP8266 at 74880 bps, Rockchip SoCs at 1.5 Mbps) and is not easy to build out of U-boot. After losing too much time fighting with such tools and cycling between them, the author considered that it was about time to address the root cause of the problem, which is that none of these tools was initially written for being used as they are used nowadays, and that if they fit the purpose so badly, it's because they're simply abused.

## Miscellaneous

This program was written by Willy Tarreau <w@1wt.eu> and is licensed under MIT license. Fixes and contributions are welcome.
