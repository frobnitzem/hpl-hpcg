---
title: "Using Infiniband"
teaching: 10
exercises: 0
math: true

questions:
- "How does infiniband hardware differ from ethernet?"

objectives:
- "Provide tools, diagnostics, and references on setting up infiniband communication."

keypoints:
- "Infiniband networks do not carry standard ethernet traffic, requiring special ."
---

Infiniband hardware devices were created to support high-performance
computing applications by "short-cutting" around the TCP/IP network
stack.  In traditional messaging, small packets are sent between
two end-points that perform a complicated connection handshake.
This involves a lot of buffering and queue
wait times, which slow things down.
Infiniband hardware provides a remote-direct memory access
mechanism for hosts to communicate with one another with fewer
requirements around acknowledgments and message sizes.

This comes at a cost.  Infiniband devices have their own
network addressing mechanism distinct from (but similar to)
ethernet MAC addresses.  Also, they require special drivers,
network configuration utilities, and API calls.

The API calls are simplest, since they are
usually handled for you by an MPI library.
Those MPI libraries mostly use either
the `ibverbs` interface or else something like
UCX built on top of ibverbs.  You'll run
into those keywords as you install MPI.


## Installing Infiniband

The other two steps are complicated.
Here are some notes that could be helpful
built by following the [RedHat Infiniband Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_infiniband_and_rdma_networks)
and [rdmamojo](https://www.rdmamojo.com/2015/01/24/verify-rdma-working/).

```
# allow users to pin unlimited memory
sudo sh -c "cat >/etc/security/limits.d/rdma.conf" <<.
@users    soft    memlock     unlimited
@users    hard    memlock     unlimited
.
# add self to the users group
sudo usermod -a -G users $USER

# setup the IB manager
sudo yum install -y opensm libibverbs-utils infiniband-diags
sudo systemctl enable --now opensm

# Testing
ibstat # locate Port GUID-s
sudo ibswitches # list switches
sudo sminfo # list sm communication info

ibv_devices
ibv_devinfo mlx4_0
```

Next, test communication between two hosts.
For example, `10.52.3.83` and `10.52.3.178`.

On one side, run

    ibv_rc_pingpong

On abother, 

    ibv_rc_pingpong 10.52.3.178


## Installing MPI

Next, installing MPI using infiniband is easy using spack:

    git clone --depth=1 https://github.com/spack/spack.git
    spack install -v mpich netmod=ucx

create a `hostfile` listing nodes, 1 per line:

```
# mpi hostfile
10.52.3.83
10.52.3.178
```

Then run a test application enabling the ucx fabric

    BIN=$BIN/bin/mpirun -n 48 -env UCX_NET_DEVICES=mlx4_0:1 -f hostfile hostname

Next, run HPL and check speed:

    spack install -v hpl ^mpich netmod=ucx # doesn't seem to activate infiniband
    spack install -v hpl ^openmpi fabrics=verbs

Remember, for each run, you should create a separately named input
and run-script,

    MPIRUN=/home/cc/spack/opt/spack/linux-centos8-haswell/gcc-8.3.1/openmpi-4.1.1-6ykqelh2aqoxk6bbmdp7o63xi33puppg/bin/mpirun
    HPL=/home/cc/spack/opt/spack/linux-centos8-haswell/gcc-8.3.1/hpl-2.3-cce3avecwhxksw6eysaklvuipvb2cdo4/bin/xhpl

    export btl_openib_allow_ib=true
    $MPIRUN -n 48 --mca btl openib,self,vader --mca btl_openib_allow_ib true --hostfile ../hostfile $HPL | tee HPL.out

{% include links.md %}
