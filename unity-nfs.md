# Dell Unity NFS Setup for OpenShift

## Prerequisites

**Dell Unity Array Requirements:**
- Unity 480 with OE version 5.0 or later
- NAS Server enabled and healthy
- Network interfaces on the NAS/production VLAN
- DNS configured for the NAS network

**OpenShift Cluster Requirements:**
- OpenShift 4.10 or later
- Cluster administrator access
- NFS client utilities available on worker nodes (nfs-utils)

## Step 1: Prepare the Unity Array

1. **Create or Select a NAS Server:**
   - Log into Dell Unisphere
   - Navigate to **Storage -> File -> NAS Servers**
   - Create a NAS Server or select an existing one
   - Assign network interfaces on the NAS network

2. **Create a File System:**
   - Navigate to **Storage -> File -> File Systems**
   - Click **Create** and choose the desired NAS Server
   - Select the storage pool
   - Set size and file system name

3. **Create an NFS Export:**
   - Navigate to the file system -> **Exports**
   - Click **Create Export**
   - Set the export path (example: `/openshift`)
   - Set the access protocol to **NFS**
   - Configure client access rules

## Step 2: Configure NFS Client Access on Unity

1. **Add NFS Client Access Rules:**
   - In the export settings, add allowed client IPs or subnets
   - Choose access level (Read/Write for OpenShift workloads)
   - Enable root access only if required (default: no root access)

2. **Confirm DNS and Networking:**
   - Verify the NAS Server has correct DNS settings
   - Ensure OpenShift worker nodes can reach the NAS IPs

## Step 3: Prepare OpenShift Nodes

1. **Check if NFS utilities are already available:**

```bash
# Test if NFS is available on nodes
oc debug node/$(oc get nodes -o jsonpath='{.items[0].metadata.name}') -- chroot /host which showmount
```

If the command returns a path (e.g., `/usr/sbin/showmount`), NFS utilities are already installed and you can skip to Step 4.

2. **If NFS utilities are not found, install them with MachineConfig:**

```bash
# Create MachineConfig to install nfs-utils
oc apply -f - <<'EOF'
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: master
  name: 99-master-nfs
spec:
  config:
    ignition:
      version: 3.2.0
    systemd:
      units:
      - enabled: true
        name: nfs-client.target
EOF
```

3. **Verify node updates:**
   ```bash
   oc get mcp
   # Wait for master pool to be updated (UPDATED=True, UPDATING=False)
   ```

## Step 4: Create StorageClass for NFS (Static)

This example uses static provisioning with pre-created exports.

```bash
cat <<EOF | oc apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: unity-nfs
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: Immediate
EOF
```

## Step 5: Create a PersistentVolume

Replace `NAS_IP` and `EXPORT_PATH` with your Unity NFS export.

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: unity-nfs-pv
spec:
  capacity:
    storage: 500Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: unity-nfs
  nfs:
    server: 192.168.1.7
    path: /srv/nfs/data
EOF
```

## Step 6: Create a PersistentVolumeClaim

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: unity-nfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 500Gi
  storageClassName: unity-nfs
EOF
```

## Step 7: Test with a Pod

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nfs-test
spec:
  containers:
  - name: app
    image: registry.access.redhat.com/ubi9/ubi-minimal
    command: ["/bin/sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: nfs-vol
      mountPath: /data
  volumes:
  - name: nfs-vol
    persistentVolumeClaim:
      claimName: unity-nfs-pvc
EOF
```

Verify the mount:
```bash
# Check if the NFS mount is accessible
oc rsh nfs-test ls -la /data

# Check disk usage on the mounted path
oc rsh nfs-test df /data

# Or check pod details
oc describe pod nfs-test | grep -A 5 "Mounts:"
```

If you see the filesystem contents and can list files, the NFS mount is working successfully.

## Step 8: (Optional) Dynamic Provisioning with NFS CSI Driver

For automatic PersistentVolume creation instead of manual static provisioning, use the NFS CSI driver.

### Install NFS CSI Driver

1. **Install via OperatorHub (recommended):**
   - Navigate to **Operators → OperatorHub**
   - Search for "NFS CSI Driver"
   - Click **Install** and select a namespace (e.g., `openshift-nfs-provisioner`)

2. **Or install via CLI:**

```bash
# Create namespace
oc create namespace openshift-nfs-provisioner

# Add Helm repository
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/

# Install NFS CSI driver
helm install nfs-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --namespace openshift-nfs-provisioner \
  --set nfs.server=192.168.1.7 \
  --set nfs.path=/srv/nfs/data
```

3. **Grant Security Context Constraint (SCC) permissions:**

```bash
# Allow the provisioner to use NFS volumes
oc adm policy add-scc-to-user hostmount-anyuid -z nfs-provisioner-nfs-subdir-external-provisioner -n openshift-nfs-provisioner
```

4. **Verify the provisioner is running:**

```bash
oc get pods -n openshift-nfs-provisioner
```

### Create StorageClass for Dynamic Provisioning

First, get the correct provisioner name from the deployment:

```bash
oc get deployment nfs-provisioner-nfs-subdir-external-provisioner -n openshift-nfs-provisioner -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="PROVISIONER_NAME")].value}' && echo
```

Then create the StorageClass with that provisioner name (typically `cluster.local/nfs-provisioner-nfs-subdir-external-provisioner`):

```bash
cat <<EOF | oc apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: unity-nfs-dynamic
provisioner: cluster.local/nfs-provisioner-nfs-subdir-external-provisioner
parameters:
  archiveOnDelete: "false"
EOF
```

### Create PVC with Dynamic Provisioning

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: unity-nfs-dynamic-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: unity-nfs-dynamic
  resources:
    requests:
      storage: 100Gi
EOF
```

The CSI driver will automatically create a PersistentVolume and subdirectory on the NFS share. Check status:

```bash
oc get pvc unity-nfs-dynamic-pvc
oc get pv | grep unity-nfs-dynamic
```

### Use Dynamic PVC in a Pod

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nfs-dynamic-test
spec:
  containers:
  - name: app
    image: registry.access.redhat.com/ubi9/ubi-minimal
    command: ["/bin/sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: nfs-vol
      mountPath: /data
  volumes:
  - name: nfs-vol
    persistentVolumeClaim:
      claimName: unity-nfs-dynamic-pvc
EOF
```

Verify the mount:

```bash
oc rsh nfs-dynamic-test ls -la /data
```

### Create a VM with NFS Storage

You can create a KubeVirt VM using the NFS StorageClass by explicitly specifying it in the dataVolumeTemplates:

```bash
cat <<EOF | oc apply -f -
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: test-nfs-vm
spec:
  dataVolumeTemplates:
  - metadata:
      name: test-nfs-vm-disk
    spec:
      storage:
        storageClassName: unity-nfs-dynamic
        accessModes:
          - ReadWriteMany
        resources:
          requests:
            storage: 30Gi
      source:
        registry:
          url: docker://quay.io/containerdisks/fedora:latest
  running: true
  template:
    spec:
      domain:
        devices:
          disks:
          - disk:
              bus: virtio
            name: datavolumedisk1
        resources:
          requests:
            memory: 2Gi
      volumes:
      - dataVolume:
          name: test-nfs-vm-disk
        name: datavolumedisk1
EOF
```

Check the VM status:

```bash
oc get vm test-nfs-vm
oc get vmi test-nfs-vm
oc get pvc | grep test-nfs-vm
```

**Note:** NFS storage works for VMs but may have slower performance than block storage (iSCSI). Use NFS when ReadWriteMany access is needed or for testing purposes.

## Notes

- NFS client utilities are pre-installed on Red Hat CoreOS, so the MachineConfig is typically not needed. Run the check in Step 3.1 first.
- **Static vs. Dynamic**: Use static provisioning (Steps 4-7) for pre-created exports. Use the NFS CSI driver (Step 8) for automatic subdirectory creation and management.
- For dynamic provisioning, use an NFS CSI driver such as the external NFS provisioner or the Dell Unity CSI driver if it supports file provisioning in your version.
- Use a dedicated NAS VLAN for production environments.
- Ensure firewall rules allow TCP 2049 between worker nodes and the Unity NAS interfaces.
- **Single-node clusters**: This example uses the `master` role because your cluster has control-plane nodes with the worker role. For multi-node clusters with separate worker pools, use `worker` instead.
