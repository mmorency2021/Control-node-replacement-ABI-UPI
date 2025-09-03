# OpenShift Control Plane Node Replacement - UPI Method

## Overview

This document provides a comprehensive procedure for replacing a control plane node in an OpenShift cluster using the User Provisioned Infrastructure (UPI) method. The procedure includes five critical parts:

1. **etcd Backup and Management**
2. **Control Plane Node Replacement with Embedded ISO**
3. **BareMetalHost and Machine Configuration**
4. **Ceph OSD Recovery**
5. **Post-Replacement Validation**

---

## Prerequisites

- Administrative access to the OpenShift cluster
- Access to the physical server or BMC for the target control plane node
- RHCOS ISO image downloaded
- Network configuration keyfile prepared
- SSH ignition configuration prepared
- HTTP server accessible from the new node for serving ignition files

### Required Files

- `eno1.nmconnection` - Network configuration keyfile
- `ssh.ign` - SSH ignition configuration
- `new_controlplane.ign` - Control plane ignition configuration
- RHCOS ISO image

---

## Part 1: etcd Backup and Preparation

### 1.1 Pre-Replacement etcd Backup

Before starting the replacement process, create a comprehensive etcd backup:

```bash
# 1. Create backup directory
mkdir -p /var/etcd-backup/$(date +%Y%m%d-%H%M%S)
BACKUP_DIR="/var/etcd-backup/$(date +%Y%m%d-%H%M%S)"

# 2. Perform etcd backup
oc debug node/<control-plane-node> -- chroot /host /usr/local/bin/cluster-backup.sh $BACKUP_DIR

# 3. Verify backup contents
ls -la $BACKUP_DIR/
```

### 1.2 Check etcd Cluster Health

```bash
# Check etcd cluster status
oc get etcd -o=jsonpath='{range .items[0].status.conditions[?(@.type=="EtcdMembersAvailable")]}{.message}{"\n"}{end}'

# List etcd members
oc rsh -n openshift-etcd etcd-<control-plane-node>
etcdctl member list -w table

# Check etcd endpoints health
etcdctl endpoint health --cluster -w table
```

### 1.3 Identify Target Member

```bash
# Get the etcd member ID for the node being replaced
MEMBER_ID=$(oc rsh -n openshift-etcd etcd-<control-plane-node> etcdctl member list | grep <target-node-name> | cut -d',' -f1)
echo "Target etcd member ID: $MEMBER_ID"
```

---

## Part 2: Control Plane Node Replacement (UPI Method)

### 2.1 Prepare Network Configuration

Create the NetworkManager keyfile for static network configuration:

```bash
# Create eno1.nmconnection file
cat > eno1.nmconnection << 'EOF'
[connection]
id=eno1
type=ethernet
interface-name=eno1
autoconnect=true

[ethernet]
mac-address=b8:ce:f6:56:3d:b2

[ipv4]
method=manual
addresses=192.168.24.89/25
gateway=192.168.24.1
dns=192.168.24.80;2600:52:7:24::80
ignore-auto-dns=true
routes=0.0.0.0/0,192.168.24.1

[ipv6]
method=manual
addresses=2600:52:7:24::89/64
gateway=2600:52:7:24::1
dns=192.168.24.80;2600:52:7:24::80
ignore-auto-dns=true
routes=::/0,2600:52:7:24::1
EOF
```

### 2.2 Prepare SSH Ignition Configuration

```bash
# Generate SSH key pair if not exists
ssh-keygen -t rsa -b 4096 -f replacement-key -N "" -C "control-plane-replacement"

# Create SSH ignition file
cat > ssh.ign << EOF
{
  "ignition": {
    "version": "3.2.0"
  },
  "passwd": {
    "users": [
      {
        "name": "core",
        "sshAuthorizedKeys": [
          "$(cat replacement-key.pub)"
        ]
      },
      {
        "name": "root",
        "sshAuthorizedKeys": [
          "$(cat replacement-key.pub)"
        ]
      }
    ]
  },
  "systemd": {
    "units": [
      {
        "name": "sshd.service",
        "enabled": true
      }
    ]
  }
}
EOF
```

### 2.3 Prepare Control Plane Ignition

```bash
# Extract the control plane ignition from existing cluster
oc extract secret/master-user-data --keys=userData -n openshift-machine-api --to=-

# Serve the ignition file via HTTP server
# Option 1: Using Python HTTP server
mkdir -p /var/www/ignition
cp master-user-data /var/www/ignition/new_controlplane.ign
cd /var/www/ignition
python3 -m http.server 9000

# Option 2: Using nginx or Apache (recommended for production)
```

### 2.4 Remove the Failed Node

```bash
# Remove the node from the cluster
oc delete node <target-node-name>

# Remove the etcd member
oc rsh -n openshift-etcd etcd-<healthy-control-plane-node>
etcdctl member remove $MEMBER_ID

# Verify member removal
etcdctl member list -w table
exit
```

### 2.5 Embed Configuration into ISO

```bash
# Step 1: Embed network configuration
coreos-installer iso network embed --keyfile eno1.nmconnection rhcos-4.18.1-x86_64-live.x86_64.iso

# Step 2: Embed SSH ignition configuration  
coreos-installer iso ignition embed --ignition-file ssh.ign rhcos-4.18.1-x86_64-live.x86_64.iso

# Verify embedded configuration
coreos-installer iso show rhcos-4.18.1-x86_64-live.x86_64.iso
```

### 2.6 Boot and Install the New Node

1. **Boot from the Custom ISO:**
   - Mount the modified ISO to the server
   - Boot the server from the ISO
   - The system will automatically configure network and SSH

2. **Install CoreOS to Disk:**

Once the system boots from the ISO and network is configured:

```bash
# SSH into the booted system
ssh -i replacement-key core@192.168.24.89

# Install CoreOS to the target disk
sudo coreos-installer install /dev/disk/by-path/pci-0000:18:00.0-scsi-0:2:1:0 \
    --insecure-ignition \
    --ignition-url=http://192.168.24.80:9000/new_controlplane.ign \
    --insecure-ignition \
    --copy-network

# Reboot the system
sudo reboot
```

### 2.7 Post-Installation Verification

```bash
# Wait for the node to come online
oc get nodes -w

# Check node status
oc get node <new-node-name>

# Verify etcd cluster health
oc get etcd -o=jsonpath='{range .items[0].status.conditions[?(@.type=="EtcdMembersAvailable")]}{.message}{"\n"}{end}'

# Check etcd member list
oc rsh -n openshift-etcd etcd-<new-node-name>
etcdctl member list -w table
etcdctl endpoint health --cluster -w table
exit
```

### 2.8 Approve Pending CSRs

```bash
# Check for pending CSRs
oc get csr

# Approve pending CSRs for the new node
oc get csr -o json | jq -r '.items[] | select(.status == {}) | .metadata.name' | xargs oc adm certificate approve

# Verify all CSRs are approved
oc get csr | grep <new-node-name>
```

---

## Part 3: BareMetalHost and Machine Configuration

### 3.1 Create BareMetalHost (BMH) Resource

After the master node replacement is complete, create a BareMetalHost resource to properly manage the new control plane node:

```yaml
apiVersion: metal3.io/v1alpha1
kind: BareMetalHost
metadata:
  name: master-3-replacement
  namespace: openshift-machine-api
  labels:
    installer.openshift.io/role: control-plane
spec:
  online: true
  bootMACAddress: "b8:ce:f6:56:3d:b2"
  bmc:
    address: "idrac-virtualmedia://192.168.24.157/redfish/v1/Systems/System.Embedded.1"
    credentialsName: master-3-bmc-secret
    disableCertificateVerification: true
  rootDeviceHints:
    deviceName: "/dev/disk/by-path/pci-0000:18:00.0-scsi-0:2:1:0"
  userData:
    name: master-user-data-managed
    namespace: openshift-machine-api
  networkData:
    name: master-3-network-config-secret
    namespace: openshift-machine-api
```

Apply the BareMetalHost resource:

```bash
oc apply -f bmh-master-3-replacement.yaml
```

### 3.2 Create Machine Resource

Create a Machine resource that represents the new control plane node:

```yaml
apiVersion: machine.openshift.io/v1beta1
kind: Machine
metadata:
  name: master-3-replacement
  namespace: openshift-machine-api
  labels:
    machine.openshift.io/cluster-api-cluster: mno-cu-q6pdq
    machine.openshift.io/cluster-api-machine-role: master
    machine.openshift.io/cluster-api-machine-type: master
spec:
  lifecycleHooks: {}
  metadata:
    labels:
      machine.openshift.io/cluster-api-cluster: mno-cu-q6pdq
      machine.openshift.io/cluster-api-machine-role: master
      machine.openshift.io/cluster-api-machine-type: master
  providerSpec:
    value:
      apiVersion: machine.openshift.io/v1alpha1
      kind: BareMetalMachineProviderSpec
      image:
        checksum: "http://192.168.24.80:9000/rhcos-4.18.1-x86_64-metal.x86_64.raw.gz.sha256sum"
        url: "http://192.168.24.80:9000/rhcos-4.18.1-x86_64-metal.x86_64.raw.gz"
      userData:
        name: master-user-data-managed
        namespace: openshift-machine-api
      networkData:
        name: master-3-network-config-secret
        namespace: openshift-machine-api
      hostSelector:
        matchLabels:
          installer.openshift.io/role: control-plane
```

Apply the Machine resource:

```bash
oc apply -f machine-master-3-replacement.yaml
```

### 3.3 Retrieve ProviderID from Machine

After the Machine resource is created and the node joins the cluster, retrieve the ProviderID:

```bash
# Get the ProviderID from the Machine resource
PROVIDER_ID=$(oc get machine master-3-replacement -n openshift-machine-api -o jsonpath='{.status.providerID}')
echo "Retrieved ProviderID: $PROVIDER_ID"

# Alternative: Get ProviderID directly from the node
NODE_NAME=$(oc get nodes -l node-role.kubernetes.io/control-plane --no-headers | grep master-3 | awk '{print $1}')
NODE_PROVIDER_ID=$(oc get node $NODE_NAME -o jsonpath='{.spec.providerID}')
echo "Node ProviderID: $NODE_PROVIDER_ID"
```

### 3.4 Update Node with ProviderID

If the node doesn't have the correct ProviderID, update it:

```bash
# Get the new node name
NEW_NODE_NAME=$(oc get nodes -l node-role.kubernetes.io/control-plane --no-headers | tail -1 | awk '{print $1}')
echo "New node name: $NEW_NODE_NAME"

# Update the node with the ProviderID from the Machine
oc patch node $NEW_NODE_NAME --type=merge -p "{\"spec\":{\"providerID\":\"$PROVIDER_ID\"}}"

# Verify the update
oc get node $NEW_NODE_NAME -o jsonpath='{.spec.providerID}'
```

### 3.5 Create Required Secrets

Create the BMC credentials secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: master-3-bmc-secret
  namespace: openshift-machine-api
type: Opaque
data:
  username: cm9vdA==  # base64 encoded "root"
  password: Y2FsdmluIA==  # base64 encoded "calvin"
```

Create the network configuration secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: master-3-network-config-secret
  namespace: openshift-machine-api
type: Opaque
stringData:
  networkData: |
    network:
      version: 2
      ethernets:
        eno1:
          dhcp4: false
          dhcp6: false
          addresses:
            - 192.168.24.89/25
            - 2600:52:7:24::89/64
          gateway4: 192.168.24.1
          gateway6: 2600:52:7:24::1
          nameservers:
            addresses:
              - 192.168.24.80
              - 2600:52:7:24::80
          match:
            macaddress: b8:ce:f6:56:3d:b2
```

Apply the secrets:

```bash
oc apply -f master-3-bmc-secret.yaml
oc apply -f master-3-network-config-secret.yaml
```

### 3.6 Validate BMH and Machine Integration

```bash
# Check BareMetalHost status
oc get bmh -n openshift-machine-api master-3-replacement

# Check Machine status
oc get machine -n openshift-machine-api master-3-replacement

# Verify the Machine has the correct ProviderID
oc get machine master-3-replacement -n openshift-machine-api -o yaml | grep providerID

# Check node association
oc get nodes -o wide | grep $(echo $PROVIDER_ID | cut -d'/' -f5)
```

---

## Part 4: Ceph OSD Recovery

### 4.1 Identify Affected OSDs

```bash
# Check if the control plane node hosts any OSDs
oc get pods -n openshift-storage -o wide | grep osd | grep <target-node-name>

# List all OSDs and their statuses
oc rsh -n openshift-storage $(oc get pods -n openshift-storage -l app=rook-ceph-tools -o name) -- ceph osd tree

# Check cluster health
oc rsh -n openshift-storage $(oc get pods -n openshift-storage -l app=rook-ceph-tools -o name) -- ceph health detail
```

### 4.2 Remove Failed OSDs

```bash
# Get the OSD IDs associated with the failed node
OSD_IDS=$(oc rsh -n openshift-storage $(oc get pods -n openshift-storage -l app=rook-ceph-tools -o name) -- ceph osd tree | grep <target-node-name> | awk '{print $1}')

# For each affected OSD, perform the removal process
for OSD_ID in $OSD_IDS; do
    echo "Processing OSD ID: $OSD_ID"
    
    # Mark OSD as out
    oc rsh -n openshift-storage $(oc get pods -n openshift-storage -l app=rook-ceph-tools -o name) -- ceph osd out $OSD_ID
    
    # Stop the OSD service (if accessible)
    oc rsh -n openshift-storage $(oc get pods -n openshift-storage -l app=rook-ceph-tools -o name) -- ceph tell osd.$OSD_ID stop
    
    # Remove OSD from cluster
    oc rsh -n openshift-storage $(oc get pods -n openshift-storage -l app=rook-ceph-tools -o name) -- ceph osd rm $OSD_ID
    
    # Remove OSD authentication key
    oc rsh -n openshift-storage $(oc get pods -n openshift-storage -l app=rook-ceph-tools -o name) -- ceph auth del osd.$OSD_ID
    
    # Remove OSD from CRUSH map
    oc rsh -n openshift-storage $(oc get pods -n openshift-storage -l app=rook-ceph-tools -o name) -- ceph osd crush remove osd.$OSD_ID
done
```

### 4.3 Prepare New OSDs on Replacement Node

```bash
# Wait for the new control plane node to be ready
oc wait --for=condition=Ready node/<new-node-name> --timeout=600s

# Check available disks on the new node
oc debug node/<new-node-name> -- chroot /host lsblk

# If using OCS/ODF operator, add the new node as a storage node
# This assumes the disks will be automatically discovered and used

# Alternatively, manually prepare OSDs if needed:
# 1. Identify available disks for OSD use
NEW_DISK="/dev/sdX"  # Replace with actual disk

# 2. Prepare the OSD using ceph-volume (if direct access is available)
oc debug node/<new-node-name> -- chroot /host ceph-volume lvm prepare --data $NEW_DISK

# 3. Activate all OSDs
oc debug node/<new-node-name> -- chroot /host ceph-volume lvm activate --all
```

### 4.4 Add New OSDs to CRUSH Map

```bash
# Get the new OSD IDs that were created
NEW_OSD_IDS=$(oc rsh -n openshift-storage $(oc get pods -n openshift-storage -l app=rook-ceph-tools -o name) -- ceph osd tree | grep <new-node-name> | awk '{print $1}')

# Add each new OSD to the CRUSH map with appropriate weight
for NEW_OSD_ID in $NEW_OSD_IDS; do
    echo "Adding OSD ID: $NEW_OSD_ID to CRUSH map"
    
    # Calculate weight based on disk size (example: 1.0 for 1TB)
    WEIGHT="1.0"  # Adjust based on actual disk size
    
    # Add OSD to CRUSH map
    oc rsh -n openshift-storage $(oc get pods -n openshift-storage -l app=rook-ceph-tools -o name) -- \
        ceph osd crush add osd.$NEW_OSD_ID $WEIGHT root=default host=<new-node-name>
done
```

### 4.5 Monitor Cluster Recovery

```bash
# Monitor cluster health during recovery
oc rsh -n openshift-storage $(oc get pods -n openshift-storage -l app=rook-ceph-tools -o name) -- ceph -w

# Check rebalancing progress
oc rsh -n openshift-storage $(oc get pods -n openshift-storage -l app=rook-ceph-tools -o name) -- ceph status

# Verify all OSDs are up and in
oc rsh -n openshift-storage $(oc get pods -n openshift-storage -l app=rook-ceph-tools -o name) -- ceph osd tree

# Check PG status
oc rsh -n openshift-storage $(oc get pods -n openshift-storage -l app=rook-ceph-tools -o name) -- ceph pg stat
```

### 4.6 Validate ODF Components

```bash
# Check OCS/ODF operator status
oc get csv -n openshift-storage

# Verify storage cluster health
oc get storagecluster -n openshift-storage

# Check all storage pods
oc get pods -n openshift-storage

# Verify storage classes are available
oc get storageclass | grep ceph
```

---

## Part 5: Post-Replacement Validation

### 5.1 Cluster Operators Validation

```bash
# Check cluster operator status
oc get co

# Wait for all operators to be available
oc wait --for=condition=Available --timeout=600s clusteroperators.config.openshift.io --all

# Check for degraded operators
oc get co | grep -v "True.*False.*False"
```

### 5.2 etcd Cluster Validation

```bash
# Verify etcd cluster health
oc rsh -n openshift-etcd etcd-<new-node-name>
etcdctl endpoint health --cluster -w table
etcdctl member list -w table

# Check etcd performance
etcdctl check perf
exit
```

### 5.3 Final Cluster Health Validation

```bash
# Final cluster health check
oc get nodes
oc get co
oc get clusterversion

# Verify all components are running
oc get pods --all-namespaces | grep -v Running | grep -v Completed
```

### 5.4 Application Workload Validation

```bash
# Check pod distribution
oc get pods --all-namespaces -o wide | grep <new-node-name>

# Verify critical workloads
oc get pods -n openshift-etcd
oc get pods -n openshift-kube-apiserver
oc get pods -n openshift-kube-controller-manager
oc get pods -n openshift-kube-scheduler
```

---

## Troubleshooting

### Common Issues and Solutions

#### 1. Network Configuration Issues

```bash
# Check network interface status
ssh -i replacement-key core@192.168.24.89
ip addr show eno1
ip route show

# Verify DNS resolution
nslookup api.<cluster-domain>
```

#### 2. Ignition Fetch Issues

```bash
# Check ignition service status
sudo systemctl status ignition-firstboot
sudo journalctl -u ignition-firstboot

# Verify HTTP server accessibility
curl -I http://192.168.24.80:9000/new_controlplane.ign
```

#### 3. etcd Join Issues

```bash
# Check etcd logs
oc logs -n openshift-etcd etcd-<new-node-name>

# Verify etcd endpoints
oc get endpoints -n openshift-etcd
```

#### 4. Certificate Issues

```bash
# Check CSR status
oc get csr | grep Pending

# Force CSR approval
oc adm certificate approve <csr-name>
```

---

## Recovery Procedures

### If Replacement Fails

1. **Restore from etcd Backup:**
   ```bash
   # Stop etcd on all nodes
   oc patch etcd cluster -p='{"spec":{"managementState":"Unmanaged"}}' --type=merge
   
   # Restore from backup
   /usr/local/bin/cluster-restore.sh $BACKUP_DIR
   
   # Restart etcd
   oc patch etcd cluster -p='{"spec":{"managementState":"Managed"}}' --type=merge
   ```

2. **Emergency etcd Recovery:**
   ```bash
   # Follow official disaster recovery procedures
   # Refer to OpenShift documentation for complete disaster recovery
   ```

---

## Best Practices

1. **Always backup etcd before starting replacement**
2. **Test the procedure in a non-production environment first**
3. **Ensure network connectivity to ignition server**
4. **Monitor cluster health throughout the process**
5. **Have rollback procedures ready**
6. **Document any deviations from standard procedure**

---

## Security Considerations

1. **Use secure ignition server (HTTPS in production)**
2. **Rotate SSH keys after replacement**
3. **Remove temporary HTTP servers**
4. **Audit access logs**
5. **Follow organizational security policies**

---

## References

- [OpenShift UPI Documentation](https://docs.openshift.com/container-platform/4.18/installing/installing_bare_metal/installing-bare-metal.html)
- [etcd Disaster Recovery](https://docs.openshift.com/container-platform/4.18/backup_and_restore/control_plane_backup_and_restore/disaster_recovery/scenario-2-restoring-cluster-state.html)
- [CoreOS Installer Documentation](https://coreos.github.io/coreos-installer/)
- [OpenShift Data Foundation Documentation](https://access.redhat.com/documentation/en-us/red_hat_openshift_data_foundation/)

---

**Document Version:** 1.0  
**Last Updated:** $(date)  
**Tested OpenShift Versions:** 4.18, 4.19  
**Target Platform:** Bare Metal UPI
