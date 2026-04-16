---
title: "OpenShift Multi-Network Pod Configuration"
date: 2026-04-15
---

# OpenShift Multi-Network Pod Configuration

When people say "OpenShift networking", they are usually talking about more than one thing. There is the default cluster network that every pod gets, and then there are secondary networks that you can attach only to the workloads that need them. That distinction matters because the right choice depends on whether you want simple pod-to-pod reachability, direct access to a physical underlay, or near line-rate access to a device.

In practice, I usually bucket the common OpenShift options like this:

- **OVN-Kubernetes default network** for the standard pod network every workload gets
- **OVN-Kubernetes secondary networks**
  - `layer2` when you want an isolated L2 domain for east-west traffic
  - `localnet` when you want that L2 domain to extend to a physical network
- **macvlan** when a pod needs a MAC on the external segment and you are comfortable with the macvlan communication model
- **bridge** when you want a pod attached through a Linux bridge-based secondary network
- **SR-IOV** when you need direct VF assignment, low overhead, and often VLAN-backed segmentation

For this post, I am focusing on **OVN-Kubernetes localnet**, because it gives a clean path to attach selected pods to an external network without changing the primary pod network model for the cluster.

The key OpenShift-specific piece in this workflow is the **NodeNetworkConfigurationPolicy (NNCP)**. The NNCP is what programs the worker node networking so OVN-Kubernetes knows which host bridge should carry the `localnet` traffic.

## Where `localnet` fits

`localnet` is an OVN-Kubernetes secondary network topology. It is useful when a pod needs:

- access to an existing physical subnet
- access to an external gateway on that subnet
- a second interface in addition to the default pod interface

Conceptually, the flow is:

1. Create a **NodeNetworkConfigurationPolicy (NNCP)** to define the host-side bridge mapping on the worker nodes.
2. Create a `NetworkAttachmentDefinition` that declares a `localnet` topology.
3. Attach a pod to that NAD with a Multus network annotation.
4. Exec into the pod and validate that the secondary interface and connectivity behave as expected.

## Prerequisites

Before applying manifests, make sure the following are true:

- The cluster is using **OVN-Kubernetes**.
- The **Kubernetes NMState Operator** is installed.
- Multus secondary networks are available.
- The worker nodes can reach the target underlay network through the bridge you map.
- If the physical network is VLAN-tagged, the switch ports connected to the workers allow that VLAN.

## Step 1: Create the namespace

```bash
oc create namespace test-net
```

## Step 2: Create the NNCP bridge mapping

Use [vlan200-nncp-ovn-localnet.yaml](/assets/manifests/localnet/vlan200-nncp-ovn-localnet.yaml) as the `NodeNetworkConfigurationPolicy` for the `localnet` bridge mapping.

Apply it:

```bash
oc apply -f assets/manifests/localnet/vlan200-nncp-ovn-localnet.yaml
oc get nncp br-ex-localnet
```

What this NNCP does:

- creates an OVN bridge mapping named `vlan200-ovn-localnet`
- maps that logical `localnet` name to `br-ex`
- makes `br-ex` the worker-side bridge used for the secondary network

The most important relationship is this one:

`localnet: vlan200-ovn-localnet`

That value must match the `physicalNetworkName` used later in the `NetworkAttachmentDefinition`.

If the NNCP is missing, the NAD alone is not enough. The pod might get the secondary attachment definition, but there will be no correct host-side mapping to the external network.

### Optional: mapping a dedicated OVS bridge

If `br-ex` is not the right bridge for your environment, the same NNCP pattern can be used with a dedicated OVS bridge instead. Use that approach carefully, because moving an active NIC under a new bridge on a worker can break connectivity if the host networking design is not planned first.

## Step 3: Create the `NetworkAttachmentDefinition`

Use [vlan200-nad-ovn-localnet.yaml](/assets/manifests/localnet/vlan200-nad-ovn-localnet.yaml) as the `NetworkAttachmentDefinition`.

Apply it:

```bash
oc apply -f assets/manifests/localnet/vlan200-nad-ovn-localnet.yaml
oc get network-attachment-definition -n test-net
```

What matters here:

- `topology: localnet` tells OVN-Kubernetes this is a localnet secondary network.
- `physicalNetworkName: vlan200-ovn-localnet` must match the `localnet` name from the NNCP bridge mapping.
- `subnets` enables OVN-managed address allocation on the secondary interface.
- `netAttachDefName` must match the namespace and NAD name exactly.

If your external network is VLAN-tagged, add a VLAN ID in the NAD:

```json
"vlanID": 100
```

That only works if the underlay path between the worker and the physical network is prepared for that VLAN.

## Step 4: Launch a test pod

Use [test-nad.yaml](/assets/manifests/localnet/test-nad.yaml) for the sample pod. It keeps the default pod network and adds a second interface from the `vlan200-ovn-localnet` NAD.

Apply it:

```bash
oc apply -f assets/manifests/localnet/test-nad.yaml
oc get pod -n test-net -w
```

Once the pod is `Running`, inspect the Multus status annotation:

```bash
oc get pod test-nad -n test-net \
  -o jsonpath='{.metadata.annotations.k8s\.v1\.cni\.cncf\.io/network-status}'
```

You should see the default network plus the `vlan200-ovn-localnet` attachment.

## Step 5: Exec into the pod and validate

Get an interactive shell:

```bash
oc exec -it -n test-net pod/test-nad -- sh
```

Inside the pod, check the interfaces:

```bash
ip addr show
ip -br addr
ip route
```

You should see:

- `eth0` for the normal cluster pod network
- `net1` for the localnet secondary network created by Multus

To inspect the secondary interface directly:

```bash
ip addr show dev net1
ip route show dev net1
ip neigh show dev net1
```

To test L2 and L3 connectivity on the localnet attachment:

```bash
ping -I net1 10.115.16.1
arping -I net1 10.115.16.1
```

Pick targets that actually exist on your external subnet. In this setup, `10.115.16.0/20` is the configured secondary network, so start with the expected gateway or another reachable host on that segment.

In a real environment that might be:

- the subnet gateway
- a legacy appliance
- a bare metal host
- a service IP reachable only from the underlay

If DNS is not part of the localnet path, test by IP first before assuming the network itself is broken.

## Useful verification commands from the cluster side

These are the commands I usually run while troubleshooting:

```bash
oc get nncp
oc describe nncp br-ex-localnet
oc get nns
oc get network-attachment-definition -n test-net
oc describe pod test-nad -n test-net
```

To see the network-status annotation in a readable form:

```bash
oc get pod test-nad -n test-net -o json | jq '.metadata.annotations["k8s.v1.cni.cncf.io/network-status"] | fromjson'
```

## Common failure points

When `localnet` does not work, the problem is usually one of these:

- The NNCP bridge mapping has not rolled out successfully to the worker nodes.
- `physicalNetworkName` in the NAD does not match the NNCP `localnet` mapping name.
- The external bridge does not actually reach the target subnet.
- The gateway IP was accidentally included in the assignable pool.
- The physical switch is not allowing the VLAN you configured in the NAD.
- The pod is running on a node that does not have the expected bridge mapping.

## Final thoughts

For most clusters, I think of the choices this way:

- use **OVN-Kubernetes layer2** when you only need isolated east-west connectivity
- use **OVN-Kubernetes localnet** when workloads need a clean path to an existing physical network
- use **macvlan** or **bridge** for simpler CNI-based attachments when OVN secondary topology is not the goal
- use **SR-IOV** when direct VF performance is the requirement

`localnet` is a strong middle ground. You keep the normal OpenShift pod networking model, but you can still attach the selected workloads that need underlay access through a second interface.

## References

- [OpenShift documentation: understanding multiple networks](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/multiple_networks/understanding-multiple-networks)
- [OVN-Kubernetes multi-homing and secondary network topologies](https://ovn-kubernetes.io/features/multiple-networks/multi-homing/)
- [Example of localnet bridge mapping and VLAN-backed NAD](https://redhatquickcourses.github.io/ocp-virt-cookbook/ocp-virt-cookbook/1/networking/localnet-vlan.html)
