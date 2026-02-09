# Dell Unity 480 iSCSI Setup for OpenShift

## Prerequisites

**Dell Unity Array Requirements:**
- Unity 480 with OE version 5.0 or later
- iSCSI service enabled
- Storage pools configured
- Management IP and iSCSI portal IPs configured

**OpenShift Cluster Requirements:**
- OpenShift 4.10 or later
- Cluster administrator access
- iSCSI initiator tools installed on worker nodes

## Step 1: Prepare the Unity Array

1. **Configure iSCSI Service on Unity:**
   - Log into Dell Unisphere
   - Navigate to **Settings → Network → Ethernet**
   - Configure iSCSI interfaces with appropriate IP addresses
   - Go to **Settings → Protocols → iSCSI** and enable iSCSI service

2. **Create Storage Pool:**
   - Navigate to **Storage → Pools**
   - Create or verify storage pool availability

3. **Create User Account for CSI:**
   - Navigate to **Settings → Access → Users**
   - Create a user with **Operator** or **Storage Administrator** role
   - Note the username and password

## Step 2: Prepare OpenShift Nodes

1. **Install iSCSI initiator on all worker nodes:**
   ```bash
   # Create MachineConfig to install iscsi-initiator-utils
   oc apply -f - <<EOF
   apiVersion: machineconfiguration.openshift.io/v1
   kind: MachineConfig
   metadata:
     labels:
       machineconfiguration.openshift.io/role: worker
     name: 99-worker-iscsi
   spec:
     config:
       ignition:
         version: 3.2.0
       systemd:
         units:
         - enabled: true
           name: iscsid.service
   EOF
   ```

2. **Verify iSCSI service status** (nodes will reboot):
   ```bash
   oc get mcp
   # Wait for worker pool to be updated
   ```

## Step 2A: Configure Dedicated Storage VLAN (Optional but Recommended)

For production environments, it's recommended to use a dedicated VLAN for storage traffic to isolate iSCSI traffic and improve performance.

### Install the Kubernetes NMState Operator

**Via OpenShift Console:**
- Navigate to **Operators → OperatorHub**
- Search for "Kubernetes NMState Operator"
- Click **Install** and use default settings

**Or via CLI:**
```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-nmstate
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-nmstate
  namespace: openshift-nmstate
spec:
  targetNamespaces:
    - openshift-nmstate
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: kubernetes-nmstate-operator
  namespace: openshift-nmstate
spec:
  channel: stable
  name: kubernetes-nmstate-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

### Deploy NMState Instance

```bash
cat <<EOF | oc apply -f -
apiVersion: nmstate.io/v1
kind: NMState
metadata:
  name: nmstate
spec: {}
EOF
```

Wait for the instance to be ready:
```bash
oc get nmstate -w
```

### Create NodeNetworkConfigurationPolicy for Storage VLAN

**For node-specific configurations** (recommended for production):

```bash
cat <<EOF | oc apply -f -
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: storage-vlan-worker-1
spec:
  nodeSelector:
    kubernetes.io/hostname: "worker-1.example.com"
  desiredState:
    interfaces:
      - name: ens192.100  # VLAN interface (parent.vlan-id)
        type: vlan
        state: up
        vlan:
          base-iface: ens192  # Your physical interface
          id: 100             # Your storage VLAN ID
        ipv4:
          enabled: true
          dhcp: false
          address:
            - ip: 192.168.100.11  # Unique per node
              prefix-length: 24
        mtu: 9000  # Jumbo frames for better performance
---
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: storage-vlan-worker-2
spec:
  nodeSelector:
    kubernetes.io/hostname: "worker-2.example.com"
  desiredState:
    interfaces:
      - name: ens192.100
        type: vlan
        state: up
        vlan:
          base-iface: ens192
          id: 100
        ipv4:
          enabled: true
          dhcp: false
          address:
            - ip: 192.168.100.12  # Unique per node
              prefix-length: 24
        mtu: 9000
---
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: storage-vlan-worker-3
spec:
  nodeSelector:
    kubernetes.io/hostname: "worker-3.example.com"
  desiredState:
    interfaces:
      - name: ens192.100
        type: vlan
        state: up
        vlan:
          base-iface: ens192
          id: 100
        ipv4:
          enabled: true
          dhcp: false
          address:
            - ip: 192.168.100.13  # Unique per node
              prefix-length: 24
        mtu: 9000
EOF
```

**Key Configuration Parameters to Customize:**
- **Physical interface**: `ens192` - Change to your actual interface name (use `oc debug node/<node>` and `ip link` to find)
- **VLAN ID**: `100` - Use your storage VLAN ID
- **Storage subnet**: `192.168.100.0/24` - Your storage network
- **Node IPs**: `192.168.100.11, .12, .13` - Assign unique IPs per node
- **Hostname**: Replace with your actual node hostnames (use `oc get nodes` to list)

### Verify VLAN Configuration

```bash
# Check policy status
oc get nncp

# Check detailed status
oc get nncp storage-vlan-worker-1 -o yaml

# Verify on nodes
oc debug node/<node-name>
chroot /host
ip addr show ens192.100
ip route
ping -c 4 192.168.100.20  # Ping Unity iSCSI portal
```

### Configure Unity Array for Storage VLAN

On your Dell Unity array:
1. Log into Unisphere
2. Navigate to **Settings → Network → Ethernet**
3. Configure iSCSI interfaces on the storage VLAN:
   - Example: 192.168.100.20 (iSCSI Portal 1)
   - Example: 192.168.100.21 (iSCSI Portal 2)
4. Ensure the VLAN tag matches (VLAN 100 in this example)

### Test iSCSI Discovery on Storage VLAN

```bash
oc debug node/<node-name>
chroot /host

# Test connectivity to Unity iSCSI portals
ping -c 4 -I ens192.100 192.168.100.20

# Discover iSCSI targets (after CSI installation)
iscsiadm -m discovery -t st -p 192.168.100.20:3260 -I ens192.100
iscsiadm -m discovery -t st -p 192.168.100.21:3260 -I ens192.100
```

### Configure Multipath for High Availability

```bash
cat <<EOF | oc apply -f -
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
  name: 99-worker-multipath-conf
spec:
  config:
    ignition:
      version: 3.2.0
    storage:
      files:
      - contents:
          source: data:text/plain;charset=utf-8;base64,$(echo -n 'defaults {
    user_friendly_names yes
    find_multipaths yes
}

devices {
    device {
        vendor "DGC"
        product ".*"
        product_blacklist "LUNZ"
        path_grouping_policy group_by_prio
        path_selector "round-robin 0"
        path_checker tur
        features "1 queue_if_no_path"
        hardware_handler "1 alua"
        prio alua
        failback immediate
        rr_weight uniform
        rr_min_io_rq 1
        no_path_retry 60
    }
}
' | base64 -w0)
        mode: 0600
        path: /etc/multipath.conf
    systemd:
      units:
      - enabled: true
        name: multipathd.service
EOF
```

Wait for nodes to reboot and apply the configuration:
```bash
oc get mcp -w
```

### Network Architecture with Storage VLAN

```
┌─────────────────────────────────────────┐
│   OpenShift Worker Node                 │
│                                          │
│  ┌────────────────┐  ┌────────────────┐│
│  │  Management    │  │  Storage VLAN  ││
│  │  ens192        │  │  ens192.100    ││
│  │  10.0.0.x      │  │  192.168.100.x ││
│  └────────┬───────┘  └────────┬───────┘│
└───────────┼──────────────────┼─────────┘
            │                   │
            │                   │ iSCSI Traffic (Isolated)
            │                   │
            │                   │
┌───────────┼──────────────────┼─────────┐
│   Dell Unity 480                        │
│                                          │
│  Management:     10.0.0.100             │
│  iSCSI Portal 1: 192.168.100.20 (VLAN)  │
│  iSCSI Portal 2: 192.168.100.21 (VLAN)  │
└──────────────────────────────────────────┘
```

**Benefits of Dedicated Storage VLAN:**
- **Isolated traffic**: Storage traffic doesn't compete with application traffic
- **Security**: Segregated network for storage communications
- **Performance**: Dedicated bandwidth for iSCSI with jumbo frames (MTU 9000)
- **QoS**: Can apply quality of service policies at network level

## Step 3: Install Dell CSI Operator

1. **Install the Dell CSI Operator for Unity from OperatorHub:**
   ```bash
   oc create namespace dell-unity-csi
   ```

2. **Via OpenShift Console:**
   - Navigate to **Operators → OperatorHub**
   - Search for "Dell CSI Operator for Unity XT"
   - Click **Install**
   - Select namespace: `dell-unity-csi`
   - Click **Install**

3. **Or via CLI:**
   ```bash
   cat <<EOF | oc apply -f -
   apiVersion: v1
   kind: Namespace
   metadata:
     name: dell-unity-csi
   ---
   apiVersion: operators.coreos.com/v1
   kind: OperatorGroup
   metadata:
     name: dell-unity-operator-group
     namespace: dell-unity-csi
   spec:
     targetNamespaces:
       - dell-unity-csi
   ---
   apiVersion: operators.coreos.com/v1alpha1
   kind: Subscription
   metadata:
     name: dell-csi-operator-unity
     namespace: dell-unity-csi
   spec:
     channel: stable
     name: dell-csi-operator-unity
     source: certified-operators
     sourceNamespace: openshift-marketplace
   EOF
   ```

## Step 4: Create Secrets for Unity Credentials

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: unity-creds
  namespace: dell-unity-csi
type: Opaque
data:
  username: $(echo -n 'your-unity-username' | base64)
  password: $(echo -n 'your-unity-password' | base64)
EOF
```

## Step 5: Create Unity CSI Driver Configuration

**Without Storage VLAN:**
```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: unity-config
  namespace: dell-unity-csi
data:
  config.yaml: |
    - arrayId: "UNITY-SERIAL-NUMBER"
      username: "your-unity-username"
      password: "your-unity-password"
      endpoint: "https://unity-management-ip"
      skipCertificateValidation: true
      isDefault: true
EOF
```

**With Storage VLAN (if you completed Step 2A):**
```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: unity-config
  namespace: dell-unity-csi
data:
  config.yaml: |
    - arrayId: "UNITY-SERIAL-NUMBER"
      username: "your-unity-username"
      password: "your-unity-password"
      endpoint: "https://unity-management-ip"  # Management IP (non-storage network)
      skipCertificateValidation: true
      isDefault: true
      iscsiPortals:
        - "192.168.100.20:3260"  # Storage VLAN IP - iSCSI Portal 1
        - "192.168.100.21:3260"  # Storage VLAN IP - iSCSI Portal 2
EOF
```

**Note:** If using a storage VLAN, the `iscsiPortals` should point to the Unity iSCSI interfaces on the storage VLAN, while the `endpoint` remains on the management network.

## Step 6: Deploy CSI Driver Instance

```bash
cat <<EOF | oc apply -f -
apiVersion: storage.dell.com/v1
kind: CSIUnity
metadata:
  name: unity
  namespace: dell-unity-csi
spec:
  driver:
    configVersion: v7
    replicas: 2
    common:
      image: "dellemc/csi-unity:v2.9.0"
      imagePullPolicy: IfNotPresent
      envs:
        - name: X_CSI_UNITY_NODENAME_PREFIX
          value: "csi-node"
        - name: X_CSI_UNITY_SYNC_NODEINFO_INTERVAL
          value: "15"
  sideCars:
    - name: provisioner
      args: ["--volume-name-prefix=csivol"]
EOF
```

## Step 7: Create Storage Class

**For iSCSI (block storage):**
```bash
cat <<EOF | oc apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: unity-iscsi
provisioner: csi-unity.dellemc.com
parameters:
  arrayId: "UNITY-SERIAL-NUMBER"
  protocol: "iSCSI"
  storagePool: "pool_name"
  thinProvisioned: "true"
  isDataReductionEnabled: "false"
  tieringPolicy: "0"
  csi.storage.k8s.io/fstype: "ext4"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
EOF
```

**For NFS (file storage):**
```bash
cat <<EOF | oc apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: unity-nfs
provisioner: csi-unity.dellemc.com
parameters:
  arrayId: "UNITY-SERIAL-NUMBER"
  protocol: "NFS"
  storagePool: "pool_name"
  thinProvisioned: "true"
  nasServer: "nas-server-name"
  hostIoSize: "8192"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: Immediate
EOF
```

## Step 8: Verify Installation

```bash
# Check operator deployment
oc get pods -n dell-unity-csi

# Check CSI driver pods
oc get pods -n dell-unity-csi | grep unity

# Check storage classes
oc get sc

# Check CSI driver
oc get csidrivers
```

## Step 9: Test with Sample PVC

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: unity-iscsi
  resources:
    requests:
      storage: 10Gi
EOF
```

```bash
# Verify PVC creation
oc get pvc test-pvc
```

## Troubleshooting Tips

1. **Check driver logs:**
   ```bash
   oc logs -n dell-unity-csi -l app=unity-controller
   ```

2. **Verify iSCSI connectivity from nodes:**
   ```bash
   oc debug node/<node-name>
   chroot /host
   iscsiadm -m discovery -t st -p <unity-iscsi-ip>
   ```

3. **Check array connectivity:**
   ```bash
   oc get events -n dell-unity-csi
   ```

4. **If using Storage VLAN, verify VLAN interface:**
   ```bash
   # Check VLAN policy status
   oc get nncp
   
   # Verify VLAN interface on node
   oc debug node/<node-name>
   chroot /host
   ip addr show ens192.100
   ping -c 4 -I ens192.100 192.168.100.20
   
   # Check multipath status
   multipath -ll
   ```

5. **Check iSCSI sessions:**
   ```bash
   oc debug node/<node-name>
   chroot /host
   iscsiadm -m session
   ```

6. **Verify CSI driver pods are running:**
   ```bash
   oc get pods -n dell-unity-csi
   oc describe pod <controller-pod-name> -n dell-unity-csi
   ```

## Important Notes

- Replace `UNITY-SERIAL-NUMBER` with your Unity array serial number
- Replace `unity-management-ip` with your Unity management IP
- Replace `pool_name` with your actual storage pool name
- Replace `ens192` with your actual physical network interface name
- For production, use proper certificates instead of `skipCertificateValidation: true`
- Ensure network connectivity between OpenShift nodes and Unity iSCSI IPs
- **Recommended**: Configure a dedicated storage VLAN (Step 2A) for production deployments
- **Recommended**: Configure multipathing for high availability (included in Step 2A)
- If using storage VLAN, ensure Unity iSCSI portals are configured on the same VLAN
- Use jumbo frames (MTU 9000) on storage VLAN for better performance

This setup uses the Dell CSI Operator, which is the recommended method for OpenShift deployments and provides automated lifecycle management of the CSI driver.
