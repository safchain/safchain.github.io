+++
date = "2015-10-17T18:04:26+02:00"
draft = false
title = "OpenContrail DevStack"
tags = ["SDN", "NFV", "OpenStack", "Neutron", "OpenContrail"]

+++

Some people asked me on IRC how to deploy an OpenContrail DevStack. So in this post I will provide two scripts in order to bootstrap such deployments, in a single node way or in a two nodes way.

Setup
-----

Some requierements and recommendations about the sizing of the VM for your installation. The build process of Opencontrail is quite long so don't hesitate to create a VM with more than one CPUi and at least 4Go of ram. I would suggest to have at least 2 network interfaces, OpenContrail can be installed with only one interface but if you plan to debug you will be more confident with an admin and user interface.

All-In-One node
---------------

Below a script that just leverages the explaination that you can find [here](https://github.com/Juniper/contrail-installer). Basically this script is creating a localrc file that the [contrail-installer](https://github.com/Juniper/contrail-installer) will use. Then it starts to build the whole solution. Finally it instanciates a devstack providing a correct local.conf. You will notice that this script does some weird stuffs like removing/upgrading some packages, this is here to fix dependencies issues.

```shell
#!/bin/bash

if [ -z "$2" ]
then
    echo "Usage: $0 <username@hostname> <interface>"
    exit -1
fi

HOST=$1
INTF=$2

# OpenContrail localrc file
LOCALRC=$(cat <<'
USE_SCREEN=True
STACK_DIR=\$(cd \$(dirname \$0) && pwd)
LOG_LEVEL=3
LOG_DIR=\$STACK_DIR/log/screens
LOGFILE=\$STACK_DIR/log/contrail.log
LOGDAYS=1

ADMIN_PASSWORD=contrail123

RABBIT_USER=stackrabbit
RABBIT_PASSWORD=\$ADMIN_PASSWORD

SERVICE_TIMEOUT=90
SERVICE_HOST=$IP

INSTALL_PROFILE=ALL

PHYSICAL_INTERFACE=$INTF

CASS_MAX_HEAP_SIZE=128M
CASS_HEAP_NEWSIZE=20M

CONTRAIL_DEFAULT_INSTALL=False

CONTRAIL_REPO_PROTO=https
NB_JOBS=4
')

# Devstack local.conf file
LOCALCONF=$(cat <<'
[[local|localrc]]
RECLONE=yes

HOST_IP=$IP

PASSWORD=contrail123
ADMIN_PASSWORD=\$PASSWORD
MYSQL_PASSWORD=\$PASSWORD
RABBIT_PASSWORD=\$PASSWORD
SERVICE_PASSWORD=\$PASSWORD
SERVICE_TOKEN=token
MULTI_HOST=1

disable_service n-net tempest heat ec2
enable_service q-svc n-cpu key
Q_PLUGIN=opencontrail

NOVA_VIF_DRIVER=nova_contrail_vif.contrailvif.VRouterVIFDriver
LIBVIRT_FIREWALL_DRIVER=" "

KEYSTONE_USE_MOD_WSGI=False

LOGFILE=\$DEST/logs/stack.sh.log
LOGDAYS=1
SCREEN_LOGDIR=\$DEST/logs/screen
')

# OpenContrail Install
ssh $1 <<EOF
    export IP=\$(ip addr | grep -E 'inet.*$INTF' | awk '{print \$2}' | cut -d '/' -f 1)
    export INTF=$INTF

    sudo apt-get install -y git
    ls contrail-installer || git clone \
        http://github.com/juniper/contrail-installer.git
    echo -e "$LOCALRC" > contrail-installer/localrc
    cd contrail-installer
    ./contrail.sh build
    ./contrail.sh build
    ./contrail.sh install
    ./contrail.sh configure
    sudo pip install "greenlet>=0.4.7" --upgrade
    ./contrail.sh start

    sleep 30
    cd ..

    ls devstack || git clone http://github.com/openstack-dev/devstack.git
    echo -e "$LOCALCONF" > devstack/local.conf
    sudo apt-get remove --purge -y python-pastedeploy
    cd devstack
    ./unstack.sh
    cp cp ../contrail-installer/devstack/lib/neutron_thirdparty/opencontrail \
        lib/neutron_thirdparty/opencontrail
    cp ../contrail-installer/devstack/lib/neutron_plugins/opencontrail \
        lib/neutron_plugins/opencontrail
    ./stack.sh
EOF
```

Compute node
------------

Below almost the same script, but this one will start what is needed for a compute node.

```shell
#!/bin/bash

if [ -z "$3" ]
then
    echo "Usage: $0 <username@hostname> <interface> <controller ip>"
    exit -1
fi

HOST=$1
INTF=$2
CTL=$3

# OpenContrail localrc file
LOCALRC=$(cat <<<'
USE_SCREEN=True
STACK_DIR=\$(cd \$(dirname \$0) && pwd)
LOG_LEVEL=3
LOG_DIR=\$STACK_DIR/log/screens
LOGFILE=\$STACK_DIR/log/contrail.log
LOGDAYS=1

ADMIN_PASSWORD=contrail123

RABBIT_USER=stackrabbit
RABBIT_PASSWORD=\$ADMIN_PASSWORD

SERVICE_TIMEOUT=90
SERVICE_HOST=$CTL
COMPUTE_HOST_IP=$IP

INSTALL_PROFILE=COMPUTE

PHYSICAL_INTERFACE=$INTF

CASS_MAX_HEAP_SIZE=128M
CASS_HEAP_NEWSIZE=20M

CONTRAIL_DEFAULT_INSTALL=False

CONTRAIL_REPO_PROTO=https
NB_JOBS=4
')

# Devstack local.conf file
LOCALCONF=$(cat <<<'
[[local|localrc]]
RECLONE=yes

HOST_IP=$IP

PASSWORD=contrail123
ADMIN_PASSWORD=\$PASSWORD
MYSQL_PASSWORD=\$PASSWORD
RABBIT_PASSWORD=\$PASSWORD
SERVICE_PASSWORD=\$PASSWORD
SERVICE_TOKEN=token
MULTI_HOST=1

NOVA_VIF_DRIVER=nova_contrail_vif.contrailvif.VRouterVIFDriver
LIBVIRT_FIREWALL_DRIVER=" "

ENABLED_SERVICES=n-cpu,neutron

SERVICE_HOST=$CTL
MYSQL_HOST=\$SERVICE_HOST
RABBIT_HOST=\$SERVICE_HOST
Q_HOST=\$SERVICE_HOST
GLANCE_HOSTPORT=\$SERVICE_HOST:9292

VNCSERVER_PROXYCLIENT_ADDRESS=\$HOST_IP
VNCSERVER_LISTEN=0.0.0.0

LOGFILE=\$DEST/logs/stack.sh.log
LOGDAYS=1
SCREEN_LOGDIR=\$DEST/logs/screen
')

# OpenContrail Install
ssh $1 <<EOF
    export IP=\$(ip addr | grep -E 'inet.*$INTF' | awk '{print \$2}' | cut -d '/' -f 1)
    export INTF=$INTF
    export CTL=$CTL

    sudo apt-get install -y git
    ls contrail-installer || git clone \
        http://github.com/juniper/contrail-installer.git
    echo -e "$LOCALRC" > contrail-installer/localrc
    cd contrail-installer
    ./contrail.sh build
    ./contrail.sh build
    ./contrail.sh install
    ./contrail.sh configure
    sudo pip install "greenlet>=0.4.7" --upgrade
    ./contrail.sh start

    sleep 30
    cd ..

    ls devstack || git clone http://github.com/openstack-dev/devstack.git
    echo -e "$LOCALCONF" > devstack/local.conf
    sudo apt-get remove --purge -y python-pastedeploy
    cd devstack
    ./unstack.sh
    cp cp ../contrail-installer/devstack/lib/neutron_thirdparty/opencontrail \
        lib/neutron_thirdparty/opencontrail
    cp ../contrail-installer/devstack/lib/neutron_plugins/opencontrail \
        lib/neutron_plugins/opencontrail
    ./stack.sh
EOF
```
