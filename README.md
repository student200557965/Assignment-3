# Assignment-3
#configure-host.sh
#!/bin/bash

VERBOSE=false
HOSTNAME=""
IP_ADDRESS=""
HOST_ENTRY_NAME=""
HOST_ENTRY_IP=""

trap '' TERM HUP INT

function verbose {
    if [ "$VERBOSE" = true ]; then
        echo "$@"
    fi
}

while [[ "$#" -gt 0 ]]; do
    case $1 in
        -verbose) VERBOSE=true ;;
        -name) HOSTNAME="$2"; shift ;;
        -ip) IP_ADDRESS="$2"; shift ;;
        -hostentry) HOST_ENTRY_NAME="$2"; HOST_ENTRY_IP="$3"; shift 2 ;;
        *) echo "Unknown parameter passed: $1"; exit 1 ;;
    esac
    shift
done

function update_hostname {
    local current_hostname=$(hostname)
    if [ "$current_hostname" != "$HOSTNAME" ]; then
        verbose "Updating hostname from $current_hostname to $HOSTNAME"
        echo "$HOSTNAME" > /etc/hostname
        hostnamectl set-hostname "$HOSTNAME"
        sed -i "s/$current_hostname/$HOSTNAME/g" /etc/hosts
        logger "Hostname changed from $current_hostname to $HOSTNAME"
    else
        verbose "Hostname is already $HOSTNAME"
    fi
}

function update_ip {
    local current_ip=$(hostname -I | awk '{print $1}')
    if [ "$current_ip" != "$IP_ADDRESS" ]; then
        verbose "Updating IP address from $current_ip to $IP_ADDRESS"
        sed -i "s/$current_ip/$IP_ADDRESS/g" /etc/hosts
        sed -i "s/address: .*/address: $IP_ADDRESS/" /etc/netplan/*.yaml
        netplan apply
        logger "IP address changed from $current_ip to $IP_ADDRESS"
    else
        verbose "IP address is already $IP_ADDRESS"
    fi
}

function update_hosts_entry {
    if ! grep -q "$HOST_ENTRY_NAME" /etc/hosts; then
        verbose "Adding host entry $HOST_ENTRY_NAME with IP $HOST_ENTRY_IP"
        echo "$HOST_ENTRY_IP $HOST_ENTRY_NAME" >> /etc/hosts
        logger "Added host entry $HOST_ENTRY_NAME with IP $HOST_ENTRY_IP"
    else
        verbose "Host entry $HOST_ENTRY_NAME already exists"
    fi
}

if [ -n "$HOSTNAME" ]; then
    update_hostname
fi

if [ -n "$IP_ADDRESS" ]; then
    update_ip
fi

if [ -n "$HOST_ENTRY_NAME" ] && [ -n "$HOST_ENTRY_IP" ]; then
    update_hosts_entry
fi


#lab3.sh
#!/bin/bash

VERBOSE=false

function verbose {
    if [ "$VERBOSE" = true ]; then
        echo "$@"
    fi
}

if [ "$1" == "-verbose" ]; then
    VERBOSE=true
fi

function check_status {
    if [ $? -ne 0 ]; then
        echo "Error encountered. Exiting."
        exit 1
    fi
}

verbose "Transferring configure-host.sh to server1-mgmt"
scp configure-host.sh remoteadmin@192.168.1.101:/root
check_status
verbose "Running configure-host.sh on server1-mgmt"
if [ "$VERBOSE" = true ]; then
    ssh remoteadmin@192.168.1.101 -- /root/configure-host.sh -verbose -name loghost -ip 192.168.16.3 -hostentry webhost 192.168.16.4
else
    ssh remoteadmin@192.168.1.101 -- /root/configure-host.sh -name loghost -ip 192.168.16.3 -hostentry webhost 192.168.16.4
fi
check_status

verbose "Transferring configure-host.sh to server2-mgmt"
scp configure-host.sh remoteadmin@192.168.1.102:/root
check_status
verbose "Running configure-host.sh on server2-mgmt"
if [ "$VERBOSE" = true ]; then
    ssh remoteadmin@192.168.1.102 -- /root/configure-host.sh -verbose -name webhost -ip 192.168.16.4 -hostentry loghost 192.168.16.3
else
    ssh remoteadmin@192.168.1.102 -- /root/configure-host.sh -name webhost -ip 192.168.16.4 -hostentry loghost 192.168.16.3
fi
check_status

verbose "Updating local /etc/hosts file"
if [ "$VERBOSE" = true ]; then
    ./configure-host.sh -verbose -hostentry loghost 192.168.16.3
    ./configure-host.sh -verbose -hostentry webhost 192.168.16.4
else
    ./configure-host.sh -hostentry loghost 192.168.16.3
    ./configure-host.sh -hostentry webhost 192.168.16.4
fi
check_status

verbose "Configuration complete"
