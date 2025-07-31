# VXCAN Network Plugin for Docker

This Docker plugin provides the ability to create VXCAN tunnels for Docker containers. It is a Rust implementation (and extension) of
https://github.com/wsovalle/docker-vxcan by Wiktor S. Ovalle Correa, which is in turn based on the work of Christain Gagneraud (https://gitlab.com/chgans/can4docker).

## Requirements

Requires that the vxcan and can-gw modules are built-in or loaded into the kernel.
```bash
sudo modprobe vxcan
sudo modprobe can-gw
```

## Available Options
**vxcan.id**: Numerical identifier of the host interface (i.e., 0 for can0, or 1 for can1). Default is 0.

**vxcan.dev**: Name of the host CAN interface (excluding numeric identifier). If the device is present (i.e., a physical CAN device) then it will be used as is; otherwise, a virtual CAN interface is created to use. Default is 'vcan'.

**vxcan.peer**: Prefix for the peer device (i.e., endpoint) to use in the container. Default is 'vcanp'. Devices in the container will be enumerated by docker as they are added to the container

## Usage

### Docker
```bash
# Create a couple Docker containers to test in separate terminals
docker run --rm -it --name a1 alpine
docker run --rm -it --name a2 alpine

# Create the network
docker network create --driver ngpbach/rustycan4docker:latest -o vxcan.dev=vcan -o vxcan.id=0 -o vxcan.peer=vxcanp rust_can1

# Connect the network to the containers
docker network connect rust_can1 a1
docker network connect rust_can1 a2

# Check that the cangw rules are present (twelve total)
cangw -L

# Check that the required interfaces are present (one vcan0, 2 vxcanXXXXXXXX)
ip link

# In the container terminals
apk add can-utils
cangen vxcanp0 # from one container
candump vxcanp0 # from the other container
cangen vcan0 # from the host

# Remove the network (after closing the containers)
docker network rm rust_can1
```

### Compose Application
docker-compose applications can make use of the plugin as well.

```yml
networks:
  canbus:
    driver: ngpbach/rustycan4docker:latest
    driver_opts:
      # Uses or creates `can3` on the host
      vxcan.dev: can
      vxcan.id: 3

      # Creates `can0` in the containers
      vxcan.peer: can
```

If you have a new enough version of compose, you can specify `vxcan.peer`
differently for each service:

```yml
services:
  app:
    networks:
      canbus:
        driver_opts:
          vxcan.peer: vcan # Creates `vcan0` in the container
```

### Plugin Installation
Install from dockerhub: https://hub.docker.com/r/ngpbach/rustycan4docker.

```bash
docker plugin install ngpbach/rustycan4docker
```

### Plugin Development
Build locally with the included build script

```
PLUGIN_NAME="ngpbach/rustycan4docker" ./docker-plugin/build-plugin.sh
docker plugin enable ngpbach/rustycan4docker
```

## FAQ

### Why can't I specify the name of the device, including the enumeration, in the container?

The docker plugin specification doesn't allow for it. The best we can do is
specify the interface's prefix, and docker will enumerate them to ensure there
are no collisions. There's a proposed extension to the compose spec that would
allow you to do this natively through docker (presumably with error checking to
avoid duplicate interfaces)

If you need deterministic names, the best you can do for now is ensure that the
interface names don't collide, and then you can assume that a 0 is appended. For
example, instead of using `vcan` as the`vxcan.peer` for all your networks, use
something like `vcan-sensors`, `vcan-controllers`, and then configure your
application to use `vcan-sensors0` and `vcan-controllers0`.

If you *really* need the names to be specifically `can0`, `can1`... and want to
ensure they're deterministic, configure the names so you get
deterministically-named 0-enumerated interfaces as suggested above, and then run
a startup script in your container to rename them.

```bash
ip link set down vcan-sensors0
ip link set vcan-sensors0 name can0
ip link set up can0
```

Docker Compose Spec Proposal:
https://github.com/compose-spec/compose-spec/blob/main/05-services.md#interface_name
