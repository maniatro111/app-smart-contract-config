# Proof-of-Concept of Configurable Smart Contract in C

This is a proof-of-concept smart contract program running on Unikraft.
It uses command line arguments, TCP networking and 9pfs filesystem mounting to configure the behavior of the smart contract.

## Setup

Apart from the current repository, you also need to clone:

* [Unikraft](https://github.com/unikraft/unikraft)
* [lwip](https://github.com/unikraft/lib-lwip)
* [newlib](https://github.com/unikraft/lib-newlib)

Make sure each repository is on the `staging` branch.

It is easiest to use a file hierarchy such as:

```
.
|-- apps/
|   `-- app-smart-contract-config/
|-- libs/
|   |-- lwip/
|   `-- newlib/
`-- unikraft/
```

If you use a file hierarhy different than the one above, then edit the `Makefile` file and update the variables to point to the correct location:
* Update the `UK_ROOT` variable to point to the location of the Unikraft clone.
* Update the `UK_LIBS` variable to point to the folder storing the library clones.

If you get a `Cannot find library` error at one of the following steps, you may need to add the `lib-` prefix to the libraries listed in the `LIBS` variable.
For instance, `$(UK_LIBS)/lwip` becomes `$(UK_LIBS)/lib-lwip`

## Configure

Configure the build by running:

```
make menuconfig
```

The basic configuration is loaded from the `Config.uk` file.
The basic configuration is loaded from the `Config.uk` file.
Then add support for 9pfs, by selecting `Library Configuration` -> `vfscore: VFS Core Interface` -> `vfscore: VFS Configuration`.
The select `Automatically mount a root filesystem (/)` and select `9pfs`.
For the `Default root device` option fill `fs0`.
You should end up with an image such as the one in `9pfs_config.png`.
Save the conifugration and exit.

As a result of this, a `.config` file is created.
A KVM unikernel image will be built from the configuration.

## Build

Build the unikernel KVM image by running:

```
make
```

The resulting image is in the `build/` subfolder, usually named `build/app-smart-contract-config_kvm-x86_64`.

## Run

The resulting image is to be loaded as part of a KVM virtual machine with a software bridge allowing outside network access to the server application.
Start the image by using the `run` script (you require `sudo` privileges):

```
$ ./run
[...]
Booting from ROM...
1: Set IPv4 address 172.44.0.2 mask 255.255.255.0 gw 172.44.0.1
en1: Added
en1: Interface is up
Powered by
o.   .o       _ _               __ _
Oo   Oo  ___ (_) | __ __  __ _ ' _) :_
oO   oO ' _ `| | |/ /  _)' _` | |_|  _)
oOo oOO| | | | |   (| | | (_) |  _) :_
 OoOoO ._, ._:_:_,\_._,  .__,_:_, \___)
                    Dione 0.6.0~6a2069e
argument is 1024
Listening on port 1024...
```

The `1024` is passed as a command line argument.

The server now listens for "commands".
These commands are files in the `mnt/` folder that will be read and processed by the smart contract program.
Passing the commands `add`, `add2`, `sub`, `sub2` will result in the program reading these files from the `mnt/` folder.

On another terminal, use `nc` to connect via TCP to the program and send the commands:

```
$ nc 172.44.0.2 1024
add
22
$ nc 172.44.0.2 1024
add2
575
$ nc 172.44.0.2 1024
sub
21
$ nc 172.44.0.2 1024
sub2
17109
```

The running program will print out a summary of received commands:

```
Received connection from 172.44.0.1:48508
Received: add
Operation: addition
Sent: 22
Received connection from 172.44.0.1:48514
Received: add2
Operation: addition
Sent: 575
Received connection from 172.44.0.1:48516
Received: sub
Operation: subtraction
Sent: 21
Received connection from 172.44.0.1:48518
Received: sub2
Operation: subtraction
Sent: 17109
```

When done, close the server / KVM machine using `Ctrl+c`.
