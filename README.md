# firecracker-vsock-snapshot-reset-bug-reproducer

Reproducer for Firecracker's VSock connection reset not working as expected.

## General Steps

1. Start Firecracker
2. Create a VM with a VSock
3. Start a VSock-over-UDS listener on the host with `socat`
4. In the guest VM, connect to the listener on the host through VSock with `socat`
5. Pause the VM and create a snapshot
6. Stop the listener on the host
7. Resume the VM

## Expected Behavior

The [Firecracker VSock docs](https://github.com/firecracker-microvm/firecracker/blob/main/docs/snapshotting/snapshot-support.md#vsock-device-limitation) state:

> Firecracker handles sending the `reset` event to the vsock driver, thus the customers are no longer responsible for closing active connections.

This would mean that the `socat` instance running inside the guest VM, which has an ongoing `read` syscall, exits due to the VSock connection being reset & the `read` syscall being interrupted.

## Actual Behavior

The `socat` instance continues running and the ongoing `read` syscall does not get interrupted/the connection reset has no effect. New `read` and `write` syscalls however fail as expected (which can be caused by e.g. pressing <kbd>Enter</kbd> in `socat`), causing an EOF & thus causing `socat` to exit as expected.

## Reproduction Guide

First, extract the rootfs and kernel included in this repository (feel free to use any other rootfs that includes `socat` & kernel with the [Firecracker `defconfig`](https://github.com/loopholelabs/drafter/blob/main/os/board/firecracker-x86_64/kernel.config) if you prefer):

```shell
mkdir -p out
cd out
tar --zstd -xvf ../assets.tar.zst
cd ..
```

Now, create an initial snapshot:

```shell
# Terminal 1
source ./control.sh
start_firecracker

# Terminal 2
source ./control.sh
create_snapshot
stop_firecracker
```

Next, resume the snapshot:

```shell
# Terminal 1
start_firecracker

# Terminal 2
resume_vm
```

Now, in a new terminal, start the VSock-over-UDS listener on the host:

```shell
socat -ddd UNIX-LISTEN:vsock.sock_28,fork STDOUT
```

Next, login to the guest in terminal 1 with username `root`, then connect to the listener on the host through VSock with `socat`:

```shell
socat -ddd - VSOCK-CONNECT:2:28
```

The listener on the host should show the following:

```console
$ socat -ddd UNIX-LISTEN:vsock.sock_28,fork STDOUT
2024/08/19 14:25:51 socat[219687] I socat by Gerhard Rieger and contributors - see www.dest-unreach.org
2024/08/19 14:25:51 socat[219687] I This product includes software developed by the OpenSSL Project for use in the OpenSSL Toolkit. (http://www.openssl.org/)
2024/08/19 14:25:51 socat[219687] I This product includes software written by Tim Hudson (tjh@cryptsoft.com)
2024/08/19 14:25:51 socat[219687] I setting option "fork" to 1
2024/08/19 14:25:51 socat[219687] I socket(1, 1, 0) -> 5
2024/08/19 14:25:51 socat[219687] I starting accept loop
2024/08/19 14:25:51 socat[219687] N listening on AF=1 "vsock.sock_28"
2024/08/19 14:25:54 socat[219687] I accept(5, {1, AF=1 "<anon>"}, 2) -> 6
2024/08/19 14:25:54 socat[219687] N accepting connection from AF=1 "<anon>" on AF=1 "vsock.sock_28"
2024/08/19 14:25:54 socat[219687] I permitting connection from AF=1 "<anon>"
2024/08/19 14:25:54 socat[219687] I number of children increased to 1
2024/08/19 14:25:54 socat[219687] N forked off child process 219704
2024/08/19 14:25:54 socat[219687] I close(6)
2024/08/19 14:25:54 socat[219687] I still listening
2024/08/19 14:25:54 socat[219687] N listening on AF=1 "vsock.sock_28"
2024/08/19 14:25:54 socat[219704] I just born: child process 219704
2024/08/19 14:25:54 socat[219704] I close(4)
2024/08/19 14:25:54 socat[219704] I close(3)
2024/08/19 14:25:54 socat[219704] I just born: child process 219704
2024/08/19 14:25:54 socat[219704] I close(5)
2024/08/19 14:25:54 socat[219704] W address is opened in read-write mode but only supports write-only
2024/08/19 14:25:54 socat[219704] N using stdout for reading and writing
2024/08/19 14:25:54 socat[219704] I resolved and opened all sock addresses
2024/08/19 14:25:54 socat[219704] N starting data transfer loop with FDs [6,6] and [1,1]
```

The dialer in the guest should show the following:

```console
# socat -ddd - VSOCK-CONNECT:2:28
2024/08/19 21:25:54 socat[363] I socat by Gerhard Rieger and contributors - see www.dest-unreach.org
2024/08/19 21:25:54 socat[363] I This product includes software developed by the OpenSSL Project for use in the OpenSSL Toolkit. (http://www.openssl.org/)
2024/08/19 21:25:54 socat[363] I This product includes software written by Tim Hudson (tjh@cryptsoft.com)
2024/08/19 21:25:54 socat[363] N reading from and writing to stdio
2024/08/19 21:25:54 socat[363] I open("/dev/vsock", 00, 0000) -> 5
2024/08/19 21:25:54 socat[363] N VSOCK CID=3
2024/08/19 21:25:54 socat[363] I close(5)
2024/08/19 21:25:54 socat[363] N opening connection to AF=40 cid:2 port:28
2024/08/19 21:25:54 socat[363] I socket(40, 1, 0) -> 5
2024/08/19 21:25:54 socat[363] N successfully connected from local address AF=40 cid:4294967295 port:3305730572
2024/08/19 21:25:54 socat[363] I resolved and opened all sock addresses
2024/08/19 21:25:54 socat[363] N starting data transfer loop with FDs [0,1] and [5,5]
```

Make sure the connection works by entering text in the guest and host terminal, which should be received on the other end respectively.

Now, pause the VM and create a snapshot:

```shell
# Terminal 2
suspend_vm
stop_firecracker
```

The listener on the host should show the following:

```shell
2024/08/19 14:28:43 socat[219704] N socket 1 (fd 6) is at EOF
2024/08/19 14:28:43 socat[219704] I poll timed out (no data within 0.500000 seconds)
2024/08/19 14:28:43 socat[219704] I shutdown(6, 2)
2024/08/19 14:28:43 socat[219704] N exiting with status 0
2024/08/19 14:28:43 socat[219687] N childdied(): handling signal 17
2024/08/19 14:28:43 socat[219687] I childdied(signum=17)
2024/08/19 14:28:43 socat[219687] I number of children decreased to 0
2024/08/19 14:28:43 socat[219687] I childdied(17): cannot identify child 219704
2024/08/19 14:28:43 socat[219687] I waitpid(): child 219704 exited with status 0
2024/08/19 14:28:43 socat[219687] I waitpid(-1, {}, WNOHANG): No child processes
2024/08/19 14:28:43 socat[219687] I childdied() finished
```

Feel free to stop the listener on the host now.

Next up, resume the VM:

```shell
# Terminal 1
start_firecracker

# Terminal 2
resume_vm
```

In terminal 1, you should see that the snapshot has resumed successfully:

```console
2024-08-19T14:29:39.242498564 [anonymous-instance:main] [DevPreview] Virtual machine snapshots is in development preview - 'load snapshot' VMM action took 17143 us.
2024-08-19T14:29:39.242513819 [anonymous-instance:fc_api] The request was executed successfully. Status code: 204 No Content.
2024-08-19T14:29:39.242523959 [anonymous-instance:fc_api] 'load snapshot' API request took 17315 us.
```

However, `socat` is still running - despite the connection that it started `read`ing from before the snapshot was suspended supposedly being `reset` by Firecracker.

Future `read`/`write`s to the connection will fail as expected, which you can trigger by pressing <kbd>Enter</kbd> in terminal 1:

```console
2024/08/19 21:30:32 socat[363] N write(5, 0x7fc0139e9000, 1) completed
2024/08/19 21:30:32 socat[363] I transferred 1 bytes from 0 to 5
2024/08/19 21:30:32 socat[363] E read(5, 0x7fc0139e9000, 8192): Socket not connected
2024/08/19 21:30:32 socat[363] N exit(1)
2024/08/19 21:30:32 socat[363] I shutdown(5, 2)
```
