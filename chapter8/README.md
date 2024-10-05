# Chapter 8 - Networking examples

## Simple XDP ping example

This isn't in the book, but I've used it a few times to show how easy it is to
use eBPF to change networking behavior.

The `ping.bpf.c` file holds a simple XDP program that checks to see if a packet
is a ping (ICMP) request, and drops it if so. The `ping.py` Python program loads
this program and attaches it to the localhost interface.

* In one terminal, start `ping`. You should see it showing a ping response being
  received every second, each one with an incrementing `icmp_seq`
  number
* While `ping` is still running in the first termina, run `ping.py` in a second
  terminal. You will see trace showing that a ping packet is detected every
  second, but if you look in the first terminal you'll see that the responses
  are no longer being received. That's because the requests are being dropped by
  the XDP program, so no response is ever generated. 
* Modify `png.bpf.c` to return XDP_PASS instead of XDP_DROP when ping packets
  are detected. Restart `ping.py` in the second terminal, and you'll see ping
  responses start being received again in the first terminal.

## XDP

The basic XDP example code discussed in the book is in hello.bpf.c. Running `make` builds this and also
uses bpftool to load the program and attach it to the `lo` interface. You can
just ping localhost to see it in action.

For the remaining examples in this chapter you'll need Docker installed in the
Lima VM (or whatever Linux machine you're using for the examples from this
book). Follow the [instructions from the Docker
documentation](https://docs.docker.com/engine/install/ubuntu/#installation-methods)
to install the `docker-ce` package. For the Lima VM, the Ubuntu version codename
is `jammy`.

## Socket filter, TC and tcpconnect examples

The `network.py` file loads a variety of other networking related examples. It uses eBPF code
from the file `network.bpf.c` and attaches them to the docker0 device. You can
comment different examples in and out to see what effect they have.

If you make changes to the eBPF code, don't forget to re-run `network.py` which
will compile, load and attach the eBPF programs to events.

### Traffic-generating container

Run a container that you can use as a source for generating ping and curl
requests that arrive at the host on the docker0 interface.

```
docker run -d --rm --name pingbox -h pingbox --env TERM=xterm-color nginxdemos/hello:plain-text
```

The network looks like this

```
Host                                                pingbox
172.17.0.1 <------------veth connection------------>172.17.0.2
            docker0                            eth0
            
        --------Traffic flowing in this direction--->
                is EGRESS for docker0 on host

        <---Traffic flowing in this direction--------
            is INGRESS for docker0 on host
```

Run `bpftool prog tracelog` to see the tracing output generated by the example
eBPF programs.

### Generate tcpconnect events and socket filter data 

The tcpconnect() eBPF program is a kprobe attached to `tcp_v4_connect` that just
generates a trace message for the tcpconnect event.

The socket_filter() eBPF program examines packets received at the socket and
traces out if they are TCP or ICMP packets. It also sends a copy of TCP packets
to user space. The Python code in network.py will displays that received data. 

You can trigger these programs by sending TCP traffic, for example:

`docker exec -it pingbox curl example.com`

### XDP example

xdp() is another example eBPF program that drops ICMP requests. Try commenting this
in and out to see that if it's enabled, these packets won't make it as far as TC.

### Have XDP and TC intercept ping requests

`docker exec -it pingbox ping 172.17.0.1`

The behaviour at TC depends which function you comment in within network.py 

If you modify the XDP program to drop ping packets, they won't reach the TC
ingress event. 

## Load balancer example

See my [lb-from-scratch](https://github.com/lizrice/lb-from-scratch) repo.