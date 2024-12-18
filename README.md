# NFS Cluster Automation Script



  *This script automates the setup of an NFS (Network File System) infrastructure, which allows file sharing across a network. It configures both an NFS server (Master node) and NFS clients (Compute nodes).*

### NFS Master Node Setup:
1. Installs NFS server and sets up directories (/storage/data, /storage/scripts).
2. Configures NFS exports to share these directories with specified clients by IP or subnet.
3. Starts and enables the NFS service to ensure it's running and starts on boot.

### NFS Client Setup:
1. Sets up passwordless SSH on compute nodes for secure management.
2. Installs NFS client and mounts the shared directories automatically.
3. Adds mount entries to /etc/fstab for persistent mounting.
4. Automates setup for multiple nodes using a list of IP addresses.


**The script supports running with ./script.sh master <SUBNET> for the master node or ./script.sh compute <NODE_IP> [<NODE_IP> ...] for multiple compute nodes. It includes error handling and logging to ensure smooth execution.**





<br>





```yml


#!/bin/bash

# Define base directory for NFS shares
BASEDIR="/storage"
# Define the IP address of the NFS Master (server)
MASTER_IP="192.168.82.100"

# Function to log messages with timestamps
function log_msg() {
    local MSG=$1
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $MSG"
}

# Function to check if a command was successful
function check_command() {
    if [[ $? -ne 0 ]]; then
        log_msg "Error: $1 failed. Exiting."
        exit 1
    fi
}

# Function to set up passwordless SSH on compute nodes
function setup_passwordless_ssh() {
    local node_ip=$1
    log_msg "Setting up passwordless SSH for ${node_ip}..."
    
    # Generate SSH key if not present
    if [[ ! -f ~/.ssh/id_ed25519 ]]; then
        ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N "" -q
        check_command "Generating SSH key"
    fi

    # Copy the public key to the remote node
    ssh-copy-id -i ~/.ssh/id_ed25519.pub root@${node_ip} -o StrictHostKeyChecking=no -o PasswordAuthentication=no
    check_command "Copying SSH key to ${node_ip}"
    
    log_msg "Passwordless SSH setup complete for ${node_ip}."
}

# Function to set up the NFS server (Master node)
function nfs_master() {
    local ip=$1
    log_msg "Setting up NFS Master node..."
    apt install nfs-kernel-server -y -qq 2>&1 >/dev/null
    check_command "NFS server installation"
    
    mkdir -p ${BASEDIR}/{data,scripts}
    chmod 750 ${BASEDIR} -R  # Secure permissions
    chown nobody:nogroup ${BASEDIR} -R  # Restrict access to system users

    # Configure exports for NFS
    printf "${BASEDIR}/data\t${ip}/255.255.255.0(rw,sync,no_root_squash,no_subtree_check)\n" > /etc/exports
    printf "${BASEDIR}/scripts\t${ip}/255.255.255.0(rw,sync,no_root_squash,no_subtree_check)\n" >> /etc/exports
    
    exportfs -ra
    systemctl restart nfs-kernel-server --now
    check_command "Restarting NFS service"
    
    systemctl enable nfs-kernel-server
    log_msg "NFS Master setup complete."
}

# Function to set up the NFS client (Compute node)
function nfs_compute() {
    local node_ip=$1
    log_msg "Setting up NFS Compute node on ${node_ip}..."

    # Create a temporary script for installation
    TEMPSCRIPT=$(mktemp)
    cat <<EOF > ${TEMPSCRIPT}
        yum install nfs-utils -y -qq 2>&1 >/dev/null
        check_command "NFS client installation"
        mkdir -p ${BASEDIR}/{data,scripts}
        echo "${MASTER_IP}:${BASEDIR}/data ${BASEDIR}/data nfs defaults 0 0" >> /etc/fstab
        echo "${MASTER_IP}:${BASEDIR}/scripts ${BASEDIR}/scripts nfs defaults 0 0" >> /etc/fstab
        mount -a
        check_command "Mounting NFS directories"
EOF

    # Start SSH agent and add SSH key
    eval `ssh-agent -s`
    ssh-add ~/.ssh/id_ed25519
    check_command "SSH setup"
    
    # Transfer the script to the compute node and execute it
    scp -o BatchMode=yes -o ConnectTimeout=2 -o StrictHostKeyChecking=no -o PasswordAuthentication=no ${TEMPSCRIPT} root@${node_ip}:/tmp/
    check_command "Copying script to ${node_ip}"
    
    ssh -o BatchMode=yes -o ConnectTimeout=2 -o StrictHostKeyChecking=no -o PasswordAuthentication=no root@${node_ip} "bash /tmp/$(basename ${TEMPSCRIPT})"
    check_command "Executing script on ${node_ip}"
    
    rm -f ${TEMPSCRIPT}
    log_msg "NFS Compute setup complete on ${node_ip}."
}

# Function to handle multiple compute nodes
function setup_compute_nodes() {
    local nodes=("$@")
    for node in "${nodes[@]}"; do
        setup_passwordless_ssh ${node}
        nfs_compute ${node}
    done
}

# Main script
if [[ "$1" == "master" ]]; then
    if [[ -z "$2" ]]; then
        echo "Usage: $0 master <SUBNET>"
        exit 1
    fi
    nfs_master $2
elif [[ "$1" == "compute" ]]; then
    shift
    if [[ "$#" -eq 0 ]]; then
        echo "Usage: $0 compute <NODE_IP> [<NODE_IP> ...]"
        exit 1
    fi
    setup_compute_nodes "$@"
else
    echo "Usage: $0 {master|compute} [arguments]"
    echo "Examples:"
    echo "  $0 master 192.168.82.0/24"
    echo "  $0 compute 192.168.82.101 192.168.82.102"
    exit 1
fi



```
  - *Save as NFS_Setup.sh*



<br>




## To run the script, follow these steps based on your system's setup:

### Prerequisites
1. Ensure SSH Key Pair: Make sure you have an SSH key pair (e.g., ~/.ssh/id_ed25519 and ~/.ssh/id_ed25519.pub) on the system from which you are running the script. The script uses passwordless SSH for communication between nodes.
  
2. NFS Installation: Ensure that the NFS server (nfs-kernel-server) and NFS client (nfs-utils) are installed on the appropriate nodes.

3. Script Permissions: Make sure the script has executable permissions. You can do this by running:
```yml
chmod +x NFS_Setup.sh
```

<br>

### Steps to Run the Script

1. Run as NFS Master (Server): If you want to set up the NFS server (master node), run the script with the master argument followed by the subnet in CIDR format.

```yml
./NFS_Setup.sh master 192.168.82.0/24              # in my case
```
  - This will set up the NFS server and configure the necessary exports for the specified subnet.

2, Run as NFS Compute Node(s): If you want to set up compute nodes, run the script with the compute argument followed by one or more IP addresses.

```yml
./NFS_Setup.sh compute 192.168.82.101 192.168.82.102     # in my case
```

  - This will set up NFS client mounts on the specified compute nodes.

3. Error Handling: The script includes error handling, logging, and SSH key setup to prevent interactive prompts and ensure smooth automated deployment.


*This setup ensures that the script can be run without manual intervention, allowing for seamless deployment across multiple nodes.*













