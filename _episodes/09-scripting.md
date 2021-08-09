---
title: "Scripting OpenStack"
teaching: 5
exercises: 20
math: true

objectives:
- "Provide tools for launching OpenStack instances on TACC."
---

For reference, here is a set of scripts useful for spinning up
servers on Chameleon Cloud from the command-line.

There are no scripts to tear-down your instances.  Instead,
this can be done on the [web-UI](https://chi.tacc.chameleoncloud.org/project/instances/)
Don't forget to also delete your leases under the
"reservations" tab.

It relies on setting up the environment as follows:

## Initial setup

1. install Chameleon's blazar (pip3 install git+https://github.com/ChameleonCloud/python-blazarclient.git@chameleoncloud/stable/train)
2. install openstack (aptitude install python3-openstackclient)
3. Login to the CHI@TACC website. The following steps refer to this site.
  1. Generate an "Application Credential" using the "Identity" section of the web GUI
     - Click the "Unrestricted Access" checkbox (needed for creating reservations from the CLI).
     - Save the credential as both `openrc.sh` and `clouds.yaml`
     - run `chmod 700 openrc.sh` and `chmod 600 clouds.yaml`
 
  2. Create a reservation using the Reservations section of the web GUI.
  3. Create an "Instance" for that reservation using the "Compute" section of the web GUI.
     - This will prompt you to generate an SSH keypair.
       Save the private key as `ssh-id.rsa` and run `chmod 600 ssh-id.rsa`
     - Name the keypair "ChameleonSSH" so that you know the name to use when accessing from scripts.
  4. Create a "Floating IP" using the "Network" section of the web GUI
  5. Use the same section to "Associate" the IP with your instance.

4. Connect to the instance via ssh:
   `ssh -i ssh-id.rsa cc@`*IP address*

At this point, you can use the web interface to close down all your servers and leases.
Alternately, you could follow the side-track below to explore
some of the setup tasks coming up.


### Side-track, exploring single-node setup

1. follow the OSE MPI Benchmarks Guide:
    - [OSU MicroBenchmarks](https://ulhpc-tutorials.readthedocs.io/en/latest/parallel/mpi/OSU_MicroBenchmarks)
    - [download](https://mvapich.cse.ohio-state.edu/download/mvapich/osu-micro-benchmarks-5.7.1.tgz)

    ```
    $ mpirun -n 48 ~/inst/libexec/osu-micro-benchmarks/mpi/collective/osu_gather

    # OSU MPI Gather Latency Test v5.7.1
    # Size       Avg Latency(us)
    1                       1.48
    2                       1.51
    4                       1.54
    8                       1.55
    16                      1.60
    32                      1.69
    64                      1.74
    128                     1.86
    256                     2.06
    512                     2.30
    1024                    3.87
    2048                    4.87
    4096                    7.45
    8192                   12.16
    16384                  23.92
    32768                  49.21
    65536                  97.32
    131072                180.46
    262144                394.33
    524288               1097.02
    1048576              2186.11

    $ mpirun -n 24 ~/inst/libexec/osu-micro-benchmarks/mpi/collective/osu_gather

    # OSU MPI Gather Latency Test v5.7.1
    # Size       Avg Latency(us)
    1                       1.14
    2                       1.18
    4                       1.20
    8                       1.21
    16                      1.27
    32                      1.28
    64                      1.33
    128                     1.41
    256                     1.56
    512                     1.73
    1024                    3.10
    2048                    3.84
    4096                    5.52
    8192                    9.71
    16384                  17.74
    32768                  34.82
    65536                  64.14
    131072                116.71
    262144                227.19
    524288                705.59
    1048576              1580.68
    ```

2. follow the HP Linpack guide:
    - [Download HPL](https://www.netlib.org/benchmark/hpl/hpl-2.3.tar.gz)

3. give up and use spack
    spack install gcc@11.1.0
    spack install hpl ^blis
    https://www.advancedclustering.com/act_kb/tune-hpl-dat-file/ ~> HPL.dat

4. HPL result:

    ```
     T/V                N    NB     P     Q               Time                 Gflops
    --------------------------------------------------------------------------------
    WR11C2R4      115200   192     6     8            3417.23             2.9826e+02
    HPL_pdgesv() start time Mon May 24 01:41:36 2021

    HPL_pdgesv() end time   Mon May 24 02:38:33 2021

    --------------------------------------------------------------------------------
    ||Ax-b||_oo/(eps*(||A||_oo*||x||_oo+||b||_oo)*N)=   2.22081836e-03 ...... PASSED
    ================================================================================

    Finished      1 tests with the following results:
                  1 tests completed and passed residual checks,
                  0 tests completed and failed residual checks,
                  0 tests skipped because of illegal input values.
    --------------------------------------------------------------------------------

    End of Tests.
    ================================================================================
    ```

5. Next, try [www.hpcg-benchmark.org](https://www.hpcg-benchmark.org/).

6. IMPORTANT: create an password-less ssh key and add it to your allowed-hosts file

       ssh-keygen -t ed25519
       cp ~/.ssh/id_ed25519.pub ~/.ssh/authorized_keys

   this will allow you to login to other nodes when you start this image.

   Also important - keep a log of the setup steps you used to create this
   image in order to share with your team, reproduce, and change later.

7. Follow the [documentation](https://chameleoncloud.readthedocs.io/en/latest/technical/images.html)
   to save a snapshot of your image when done (using `sudo cc-snapshot <image_name>`).


## Makefile

This Makefile stores the series of steps needed to start
all the infrastructure needed to launch a set of servers.

It is intended to be used for launching a single node
on which software is installed.  Later, this node can be
packed into a disk image and launched across several server machines.

Obviously, running the commands here requires that their
directory is in your PATH.

```
# Makefile
NAME = install
MINUTES = 120 # 2 hours

default: $(NAME)-server.ip


$(NAME)-server.ip: $(NAME)-server.json $(NAME)-ip.json
        associate_ip.sh $(NAME)-server

$(NAME)-server.json: $(NAME)-lease.json
        start_server.sh $(NAME)-lease $(NAME)-server

$(NAME)-ip.json:
        lease_ip.sh $(NAME)-ip $(MINUTES)

$(NAME)-lease.json:
        lease_server.sh $(NAME)-lease 1 $(MINUTES)

clean:
        rm -f trial-*.json
```

Note: if you have an existing lease, create its json using
`bin/activate_lease.sh <name of lease>`.  Then, following
the Makefile above, rename it to `install-lease.json`.
Then you can use the above Makefile with your existing lease.


## Individual commands

Each of the commands below sets up one piece of server
infrastructure by launching the appropriate
OpenStack command.  Several make use of the following
python script to read values from a json-file:

```
#!/usr/bin/env python3
# read_val.py

import json

def first(x):
    def run(test):
        for xi in x:
            if test(xi):
                return xi
    return run

def main(argv):
    assert len(argv) == 3, f"Usage: {argv[0]} <file.json> <key>"
    with open(argv[1], 'r', encoding='utf-8') as f:
        x = json.loads( f.read() )

    if isinstance(x, dict):
        local = x
    else:
        local = {'x': x, 'first': first(x)}
    print( eval(argv[2], {'json':json}, local) )

if __name__=="__main__":
    import sys
    main(sys.argv)
```

```
#!/bin/bash
# ../bin/activate_lease.sh
# Read the status of a lease object until it is active.
# Store the result at <lease name>.json
# Exits with an error-code if the lease object is missing
# or not active within 30 seconds.

set -e

if [ $# -ne 1 ]; then
    echo "Usage: $0 <name>"
    exit 1
fi

name="$1"

# openstack server show -f json interactive-server

# poll until lease is active
for((i=0;i<30;i++)); do
    blazar lease-show -f json "$name" >"$name.json"
    status=$(read_val.py "$name.json" "status")
    [[ x"$status" == x"ACTIVE" ]] && break
    sleep 1
done
if [[ x"$status" != x"ACTIVE" ]]; then
    echo "Lease not activated."
    exit 1
fi
```

```
#!/bin/bash
# ../bin/activate_server.sh
# Read the status of a server object until it is active.
# Store the result at <server name>.json
# Exits with an error-code if the server object is missing
# or not active within 600 seconds (10 minutes).

set -e

if [ $# -ne 1 ]; then
    echo "Usage: $0 <name>"
    exit 1
fi

name="$1"

# poll until active
for((i=0;i<60;i++)); do
    openstack server show -f json "$name" >"$name.json"
    status=$(read_val.py "$name.json" "status")
    echo "Server Status = $status"
    [[ x"$status" == x"ACTIVE" ]] && break
    sleep 10
done
if [[ x"$status" != x"ACTIVE" ]]; then
    echo "Server not activated."
    exit 1
fi
```

```
#!/bin/bash
# ../bin/associate_ip.sh
# Associate a leased, floating IP with a server

set -e

# FIXME: there's no way to go from an IP reservation
# name to an actual IP address!
if [ $# -ne 1 ]; then
    echo "Usage: $0 <server-name>"
    exit 1
fi

#lease="$1"
server="$1"

if [ ! -s "$server.json" ]; then
    if [ -s "$server-1.json" ]; then
        server="$server-1"
    else
        echo "File not found: $server.json"
        exit 1
    fi
fi
server_id=$(read_val.py "$server.json" id)

iplist=`mktemp`
openstack floating ip list -f json >>$iplist
ip_addr=$(read_val.py "$iplist" 'first(lambda z: z["Port"] is None)["Floating IP Address"]')
rm -f $iplist

echo "Associating $ip_addr to $server"
#openstack floating ip add "$ip_addr" "$server_id"
#openstack floating ip set --port "$server_id" "$ip_addr"
openstack server add floating ip "$server_id" "$ip_addr"

echo "$ip_addr" >"$server.ip"
```

```
#!/bin/bash
# ../bin/lease_ip.sh
# Create a floating IP lease for the next <n> minutes.

set -e

if [ $# -ne 2 ]; then
    echo "Usage: $0 <name> <minutes>"
    exit 1
fi

name="$1"
n="$2"

start=$(TZ="UTC" date --date 'now' "+%Y-%m-%d %H:%M")
end=$(TZ="UTC" date --date "now+$n minutes" "+%Y-%m-%d %H:%M")
echo "Creating IP lease from $start to $end (UTC)"

PUBLIC_NETWORK_ID=$(openstack network show public -c id -f value)
blazar lease-create \
  --reservation resource_type=virtual:floatingip,network_id=${PUBLIC_NETWORK_ID},amount=1 \
  --start-date "$start" \
  --end-date "$end" \
  "$name"

activate_lease.sh "$name"
```

```
#!/bin/bash
# ../bin/lease_server.sh
# Create a node lease for the next <n> minutes.

set -e

if [ $# -ne 3 ]; then
    echo "Usage: $0 <name> <number> <minutes>"
    exit 1
fi

name="$1"
n="$2"
t="$3"

if [ $n -le 0 ]; then
    echo "Invalid number: $n"
    exit 1
fi
if [ $t -le 0 ]; then
    echo "Invalid time: $t"
    exit 1
fi

start=$(TZ="UTC" date --date 'now' "+%Y-%m-%d %H:%M")
end=$(TZ="UTC" date --date "now+$t minutes" "+%Y-%m-%d %H:%M")
echo "Creating node lease from $start to $end (UTC)"

# --physical-reservation min=$n,max=$n,resource_properties='["and", ["=", "$infiniband", "True"], [">=", "$gpu.gpu_count", "1"]]' \
# --physical-reservation min=$n,max=$n,resource_properties='["==","$node_name","c01-18"]' \
blazar lease-create \
  --start-date "$start" \
  --physical-reservation min=$n,max=$n,resource_properties='["=", "$node_type", "compute_haswell"]' \
  --end-date "$end" \
  "$name"

# Notes:
# - directly read status from lease
#   status=$(blazar lease-show "$name" -c status -f value)
# - extend an existing lease (TODO)
#   blazar lease-update --end-date "$end" "$name"
# - list all floating IP-s
#   openstack floating ip list
# - list all servers
#   openstack server list

activate_lease.sh "$name"
```

```
#!/usr/bin/env python3
# ../bin/read_val.py

import json

def first(x):
    def run(test):
        for xi in x:
            if test(xi):
                return xi
    return run

def main(argv):
    assert len(argv) == 3, f"Usage: {argv[0]} <file.json> <key>"
    with open(argv[1], 'r', encoding='utf-8') as f:
        x = json.loads( f.read() )

    if isinstance(x, dict):
        local = x
    else:
        local = {'x': x, 'first': first(x)}
    print( eval(argv[2], {'json':json}, local) )

if __name__=="__main__":
    import sys
    main(sys.argv)
```

```
#!/bin/bash
# ../bin/start_server.sh
# Start a server on the given reservation.

set -e

if [ $# -ne 2 ]; then
    echo "Usage: $0 <lease-name> <insance-name>"
    exit 1
fi

lease="$1"
instance="$2"

res_id=$(read_val.py "$lease.json" "json.loads(reservations)['id']")
min=$(read_val.py "$lease.json" "json.loads(reservations)['min']")
max=$(read_val.py "$lease.json" "json.loads(reservations)['max']")
#blazar lease-show interactive-lease

net_id=$(openstack network list --name sharednet1 -c ID -f value)

openstack server create \
  --image CC-CentOS8 \
  --flavor baremetal \
  --key-name ChameleonSSH \
  --nic net-id="$net_id" \
  --hint reservation="$res_id" \
  --min $min \
  --max $max \
  "$instance"

if [ $min -eq 1 ]; then
  activate_server.sh "$instance"
else
  for((i=1;i<=min;i++)); do
    activate_server.sh "$instance-$i"
  done
  touch "$instance.json"
fi
```

## Example Configurations

```
#!/usr/bin/env bash
# openrc.sh, mostly generated by Chameleon Cloud.

export OS_AUTH_TYPE=v3applicationcredential
export OS_AUTH_URL=https://chi.tacc.chameleoncloud.org:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_REGION_NAME="CHI@TACC"
export OS_INTERFACE=public
export OS_APPLICATION_CREDENTIAL_ID=long-string-with-the-credential
export OS_APPLICATION_CREDENTIAL_SECRET=long-string-with-the-secret
export PATH=$PATH:$PWD/bin
```

