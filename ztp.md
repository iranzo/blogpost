Table of contents:

<!-- TOC depthfrom:1 orderedlist:false -->

- [Telco 5G Zero Touch Provisioning (ZTP)](#telco-5g-zero-touch-provisioning-ztp)
  - [So... what is ZTP?](#so-what-is-ztp)
  - [How to deploy ZTP connected?](#how-to-deploy-ztp-connected)
  - [Manifest creation](#manifest-creation)
    - [Hub cluster basic elements creation](#hub-cluster-basic-elements-creation)
      - [ClusterImageSet](#clusterimageset)
      - [AssistedServiceConfig](#assistedserviceconfig)
      - [AgentServiceConfig](#agentserviceconfig)
      - [Private Key](#private-key)
      - [Pull Secret](#pull-secret)
    - [Spoke Cluster definition](#spoke-cluster-definition)
      - [AgentClusterInstall](#agentclusterinstall)
      - [SNO Cluster Definition](#sno-cluster-definition)
      - [Multi-Node Cluster Definition](#multi-node-cluster-definition)
      - [ClusterDeployment](#clusterdeployment)
      - [NMState Config](#nmstate-config)
      - [InfraEnv](#infraenv)
    - [Spoke cluster deployment](#spoke-cluster-deployment)
      - [Fully Automated ZTP](#fully-automated-ztp)
      - [Manual Spoke cluster deployment](#manual-spoke-cluster-deployment)
  - [Wrap up](#wrap-up)

<!-- /TOC -->

# Telco 5G Zero Touch Provisioning (ZTP)

Telco 5G has become a big effort in current Telco operations and deployments, where moving from traditional hardware-based equipment to Software-Defined Networking and Appliances has just become the de-facto standard.

5G is not only about the speed as in bandwidth, but also as the latency, and one of the key areas being pursued this, is getting the network functions closest to the end-user, and, one of the biggest challenges and promises, is that it will become the network for the Internet of Thing (IoT) devices, supporting a broader number of devices via technologies like IPv6, new bands for the radio, as well as smart antennas that will focus the beams towards the devices accessing the network (beamforming).

Nobody said that this could be simple, as it demands a lot of technology and instead of being datacenter-centric, the promises of higher speed and reduced latency, come with a cost: services should be closes to the customer to satisfy them.

Traditionally, the requirements for applications and services made it feasible to be on a datacenter: redundancy, high-speed network, etc., which is completely opposite to what could be expected on the user end.

For the 5G and Edge scenarios, the reality is the opposite: probably slow or even disconnected from data center environments, which still require similar approaches as the ones used in datacenters, but still having predictable downtimes, management, etc.

Telco 5G also came with some acronyms and terminology:

- ACM: Advanced Cluster Management
- IPI: Installer-Provisioned Infrastructure
- NFV: Network Functions Virtualization
- PTP: Precision Time Protocol
- RAN: Radio Access Networks
- RWN: Remote Worker Nodes
- SDN: Software-Defined Networking
- SNO: Single Node OpenShift
- UPI: User-Provisioned Infrastructure
- ZTP: Zero Touch Provisioning

Let's learn about ZTP in the next section.

## So... what is ZTP?

ZTP, as indicated above, is the acronym for Zero Touch Provisioning, a project that deploys and delivers OpenShift 4 clusters in an architecture named HUB-Spoke, where a unique Hub cluster is able to manage many Spoke clusters.

The Hub cluster will use Red Hat Advanced Cluster Management to manage and deploy the Spoke clusters.

ZTP, of course, can have two scenarios, connected and disconnected, wether the OpenShift Container Platform Worker nodes can directly access the internet or not.

Let's check the installation flow:

![](connected.png)

For comparing, in the case of a disconnected ZTP flow, many prerequisites will be required, like creating an internal registry that gets mirrored content from the deployment servers, etc. We'll be publishing another blog post with fully disconnected flow.

Let's learn how to deploy ZTP in the next section.

## How to deploy ZTP connected?

First, we will need to deploy the Red Hat Advance Cluster Manager Operator version `2.3.0` or higher from the OperatorHub:

![Operator Hub visualization](operatorhub.png)

Once we click on ACM, this window will open with the details:

![ACM Details](details.png)

There, we select the channel `release-2.3`:

![ACM install screen](install.png)

Once the Operator is installed, the prerequisites for deploying ACM will be already fulfilled, so we're ready to configure it.

Let's double-check by issuing some commands (ensure `KUBECONFIG` environment is loaded):

```sh
oc get HiveConfig -o yaml
```

Which should show something like this:

```yaml
apiVersion: hive.openshift.io/v1
kind: HiveConfig
metadata:
  name: hive
  namespace: open-cluster-management
spec:
  featureGates:
    custom:
      enabled:
        - AlphaAgentInstallStrategy
    featureSet: Custom
  logLevel: debug
  targetNamespace: hive
```

**NOTE**: If this is not the same content as you have already in your `HiveConfig` CR, please ensure that you apply this manifests, if not another CRD called `ClusterDeployment` will fail in future steps.

## Manifest creation

Here we will see the different manifest creation that we'll be using for
the deployment.

### Hub cluster basic elements creation

Even if this creation could be done in an automated way, we want to explore the different steps to gain a better understanding of the full procedure.

#### ClusterImageSet

This manifest should contain a reachable OpenShift version that will be pulled from Hive and Assisted Installer in order to deploy a Spoke cluster, and this is how it looks like:

```yaml
apiVersion: hive.openshift.io/v1
kind: ClusterImageSet
metadata:
  name: openshift-v4.8.0
  namespace: open-cluster-management
spec:
  releaseImage: quay.io/openshift-release-dev/ocp-release:4.8.0-fc.8-x86_64
```

#### AssistedServiceConfig

This is an optional `ConfigMap` that can be used to customize the Assisted Service pod deployment using annotations in the Operand (we will go deep into this topic later).

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: assisted-service-config
  namespace: open-cluster-management
  labels:
    app: assisted-service
data:
  CONTROLLER_IMAGE: quay.io/ocpmetal/assisted-installer-controller@sha256:93f193d97556711dce20b2f11f9e2793ae26eb25ad34a23b93d74484bc497ecc
  LOG_LEVEL: "debug"
```

**NOTE**: We don't recommend using this functionality in production environments and it's unsupported.

#### AgentServiceConfig

This is the Operand, the Assisted Service pod that handles Spoke cluster deployments:

```yaml
apiVersion: agent-install.openshift.io/v1beta1
kind: AgentServiceConfig
metadata:
  name: agent
  namespace: open-cluster-management
  ### This is the annotation that injects modifications in the Assisted Service pod
  annotations:
    unsupported.agent-install.openshift.io/assisted-service-configmap: "assisted-service-config"
###
spec:
  databaseStorage:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 40Gi
  filesystemStorage:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 40Gi
  ### This is a ConfigMap that only will make sense on Disconnected environments
  mirrorRegistryRef:
    name: "lab-index-mirror"
  ###
  osImages:
    - openshiftVersion: "4.8"
      version: "48.84.202106070419-0"
      url: "https://releases-rhcos-art.cloud.privileged.psi.redhat.com/storage/releases/rhcos-4.8/48.84.202106070419-0/x86_64/rhcos-48.84.202106070419-0-live.x86_64.iso"
      rootFSUrl: "https://releases-rhcos-art.cloud.privileged.psi.redhat.com/storage/releases/rhcos-4.8/48.84.202106070419-0/x86_64/rhcos-48.84.202106070419-0-live-rootfs.x86_64.img"
```

#### Private Key

This is a Secret created that contains the private key that will be used by Assisted Service pod.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: assisted-deployment-ssh-private-key
  namespace: open-cluster-management
stringData:
  ssh-privatekey: |-
    -----BEGIN OPENSSH PRIVATE KEY-----
    b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABFwAAAAdzc2gtcn
    xe+m0Tg8fgoEdogPgx6W0T30Y9b1lytPZKr3gBhKdEGD79by/vjIulP2CqkeNBCfdmIHts
    ...
    ...
    ...
    KnSXpjTZqJen9KAoSl9+U6hJ9mh8uBKAT4B74g4JtjILKiXiKkyWI75PpWb05RXxBxzUYX
    4qqJ4OPv/pnjM7UAAAAXcm9vdEBxY3QtZDE0dTExLnZsYW4yMDgBAgME
    -----END OPENSSH PRIVATE KEY-----
type: Opaque
```

#### Pull Secret

This is a Secret that contains the access credentials for the Registry access (Internal or External)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: assisted-deployment-pull-secret
  namespace: open-cluster-management
stringData:
  .dockerconfigjson: '{"auths":{"registry.ci.openshift.org":{"auth":"dXNlcjiZ3dasdNTSFffsafzJubE80LVYngtMlRGdw=="},"registry.svc.ci.openshift.org":{"auth":"dasdaddjo3b1NwNlpYX2kyVLacctNcU9F"},"quay.io":{"auth":"b3BlbnNoaWZ0LXJlbGGMVlTNkk1NlVQUQ==","lab-installer.lab-net:5000":{"auth":"ZHVtbXk6ZHVtbXk=","email":"jhendrix@karmalabs.com"}}}'
```

From this point, we will cover the creation of manifests for the Spoke clusters. As a reminder, the Hub Cluster is an unique cluster that will manage all the Spoke clusters (1:N relationship) managed by it.

### Spoke Cluster definition

In the Manifest creation phase, we still need to define the relevant CR's that will represent our Spoke clusters.

#### AgentClusterInstall

This is one of the most important elements to define, being the kind of deployment the first one to decide: SNO or Multi-Node.

#### SNO Cluster Definition

For Single-Node OpenShift we need to specify the next values:

- `spec.provisionRequirements.controlPlaneAgents`: Set to `1`, this means that we just want a ControlPlane based on 1 Master node.
- `spec.imageSetRef`: This will reference the `ClusterImageSet` created in previous steps, so ensure that those are related between them using the name.
- `spec.clusterDeploymentRef.name`: This represents the name of our ClusterDeployment, which will be created in the next step, so just catch the name and reflect it here.
- `spec.networking.clusterNetwork` and `spec.networking.serviceNetwork` are references to internal communication, so ensure that there is no overlap between them.
- `spec.networking.machineNetwork.cidr`: Represents the network range for external communication, so make sure same range as your node will use to communicate with outside is used.

**NOTE**: We **DON'T** need the API and Ingress VIP for this kind of cluster, Assisted Service will figure it out using the `spec.networking.machineNetwork.cidr` element in the CR.

This is an example of how it should look like on an IPv6 environment:

```yaml
apiVersion: extensions.hive.openshift.io/v1beta1
kind: AgentClusterInstall
metadata:
  name: lab-cluster-aci
  namespace: open-cluster-management
spec:
  clusterDeploymentRef:
    name: lab-cluster
  imageSetRef:
    name: openshift-v4.8.0
  networking:
    clusterNetwork:
      - cidr: "fd01::/48"
        hostPrefix: 64
    serviceNetwork:
      - "fd02::/112"
    machineNetwork:
      - cidr: "2620:52:0:1302::/64"
  provisionRequirements:
    controlPlaneAgents: 1
  sshPublicKey: "ssh-rsa adasdlkasjdlklaskdjadoipjasdoiasj root@xxxxXXXXxxx"
```

#### Multi-Node Cluster Definition

For Multi-Node OpenShift we need to specify the next values:

- `spec.provisionRequirements.controlPlaneAgents`: Set to `3`, this means that we just want a ControlPlane based on 3 Master nodes.
- `spec.imageSetRef`: This will reference the ClusterImageSet created in previous steps, so ensure that those are related between them using the name.
- `spec.clusterDeploymentRef.name`: This represents the name of our ClusterDeployment, will be created in the next step, so just catch the name and reflect it here.
- `spec.networking.clusterNetwork` and `spec.networking.serviceNetwork` are references to internal communication, so ensure that there is no overlap between them.
- `spec.apiVIP` and `spec.ingressVIP`: These elements reference the API and Ingress addresses for the OpenShift Spoke cluster, ensuring that they are not the same as your hub cluster.

**NOTE**: We **DON'T** need the `spec.networking.machineNetwork.cidr` for this kind of cluster, Assisted Service will figure it out using the `spec.apiVIP` and `spec.ingressVIP` elements in the CR.

This is an example of how it should look like on an IPv6 environment:

```yaml
apiVersion: extensions.hive.openshift.io/v1beta1
kind: AgentClusterInstall
metadata:
  name: lab-cluster-aci
  namespace: open-cluster-management
spec:
  clusterDeploymentRef:
    name: lab-cluster
  imageSetRef:
    name: openshift-v4.8.0
  apiVIP: "2620:52:0:1302::3"
  ingressVIP: "2620:52:0:1302::2"
  networking:
    clusterNetwork:
      - cidr: "fd01::/48"
        hostPrefix: 64
    serviceNetwork:
      - "fd02::/112"
  provisionRequirements:
    controlPlaneAgents: 3
  sshPublicKey: "ssh-rsa adasdlkasjdlklaskdjadoipjasdoiasj root@xxxxXXXXxxx"'
```

**NOTE**: This is also the CR where you will diagnose the issues in the deployment when you trigger it up, more concretely in the field `status.conditions` are all the relevant fields on the deployment status side.

#### ClusterDeployment

This element represents a Cluster as you used to see on the SaaS UI of Assisted Installer, and it's referenced from the `AgentClusterInstall` CR that contains the details regarding addressing, name, nodes, etc… Also this CR belongs to Hive API as an extension, it will need to have access to a `FeatureGate` mentioned in a past section called `AlphaAgentInstallStrategy`, so be sure that it's already enabled, if not you can do it using this patch:

```sh
oc patch hiveconfig hive --type merge -p '{"spec":{"targetNamespace":"hive","logLevel":"debug","featureGates":{"custom":{"enabled":["AlphaAgentInstallStrategy"]},"featureSet":"Custom"}}}'
```

The most important fields on this CR are the next ones:

- `spec.baseDomain` and `spec.clusterName`: Which represents the ClusterName and the Domain for it, so in this case, for example, the API address will be something like `api.lab-spoke.alklabs.com` and the ingress will be represented with something like `\*.apps.lab-spoke.alklabs.com`.
- `spec.clusterInstallRef`: Represents the reference to the AgentClusterInstall created before, so be sure you are pointing to the right name.
- `spec.pullSecretRef.name`: Points to the PullSecret's name created in the section before this one.

This is a sample as how should looks like on a IPv6 environment

```yaml
apiVersion: hive.openshift.io/v1
kind: ClusterDeployment
metadata:
  name: lab-cluster
  namespace: open-cluster-management
spec:
  baseDomain: alklabs.com
  clusterName: lab-spoke
  controlPlaneConfig:
    servingCertificates: {}
  installed: false
  clusterInstallRef:
    group: extensions.hive.openshift.io
    kind: AgentClusterInstall
    name: lab-cluster-aci
    version: v1beta1
  platform:
    agentBareMetal:
      agentSelector:
        matchLabels:
          bla: "aaa"
  pullSecretRef:
    name: assisted-deployment-pull-secret
```

#### NMState Config

This is an optional configuration that you want to add when the Network configuration needs some adjustments like work with Bonding or use a concrete VLAN or just declaring a Static IP. The NMState it's a generic/standard configuration that could be used in a separate way of Assisted Installer/ACM and the documentation can be found here and here are some examples. One NMState profile will map in a relation of 1-1 to an InfraEnv (we will cover this one later) and this profile should cover all nodes involved on the cluster.

This is a sample as to how should it look like on an IPv6 environment

```yaml
apiVersion: agent-install.openshift.io/v1beta1
kind: NMStateConfig
metadata:
  name: assisted-deployment-nmstate-lab-spoke
  labels:
    cluster-name: nmstate-lab-spoke
spec:
  config:
    interfaces:
      - name: bond99
        type: bond
        state: up
        ipv6:
          address:
          - ip:2620:52:0:1302::100
            prefix-length: 64
          enabled: true
        link-aggregation:
          mode: balance-rr
          options:
            miimon: '140'
          slaves:
          - eth0
          - eth1
  interfaces:
    - name: "eth0"
      macAddress: "02:00:00:80:12:14"
    - name: "eth1"
      macAddress: "02:00:00:80:12:15"
```

#### InfraEnv

When we create this CR, we're instructing Assisted Service to create the final ISO to be mounted on the destination nodes, so this is the final step of the Manifest Creation phase.

The most important fields on this CR are the next:

- `spec.clusterRef.name` and `spec.clusterRef.namespace`: This will reference the Name and the Namespace of our ClusterDeployment CR and where it is located, so be sure that you are pointing to the right ones.
- `spec.agentLabelSelector.matchLabels`: It should be the same created previous CR's.
- `spec.pullSecretRef.name`: Should point to the right PullSecret created in the previous stage.
- `spec.sshAuthorizedKey`: This is required in case of troubleshooting as it will allow us to login into the node and diagnose why it's not pulling the image or other errors.
- `spec.ignitionConfigOverride`: Optional, This is used if it's needed to include modifications in the Discovery ISO ignition.
- `spec.nmStateConfigLabelSelector`: Optional, This will make the relationship between the `NMState` manifest you had created on the previous step and the InfraEnv that will trigger the ISO creation on the Assisted Service pod.

This is an example of how it should look like:

```yaml
apiVersion: agent-install.openshift.io/v1beta1
kind: InfraEnv
metadata:
  name: lab-env
  namespace: open-cluster-management
spec:
  clusterRef:
    name: lab-cluster
    namespace: open-cluster-management
  sshAuthorizedKey: "ssh-rsa adasdlkasjdlklaskdjadoipjasdoiasj root@xxxxXXXXxxx"
  agentLabelSelector:
    matchLabels:
      bla: "aaa"
  pullSecretRef:
    name: assisted-deployment-pull-secret
  ignitionConfigOverride: '{"ignition": {"version": "3.1.0"}, "storage": {"files": [{"path": "/etc/someconfig", "contents": {"source": "data:text/plain;base64,aGVscGltdHJhcHBlZGluYXN3YWdnZXJzcGVj"}}]}}'
  nmStateConfigLabelSelector:
    matchLabels:
      cluster-name: nmstate-lab-spoke
```

### Spoke cluster deployment

To achieve this part we need to ensure (again) that the Hub cluster it's based on IPI and has the Metal³ pods.

To validate just execute the following command, if the output it's empty, this process will follow the manual approach, if not, the process to follow will be the real ZTP.

```sh
oc get pod -A | grep metal3
```

#### Fully Automated ZTP

This first flow will be orchestrated from Ironic and Metal3 containers and the CRD involved will be the BareMetalHost. As a pre-step we need to ensure that Metal3 pod can check other namespaces and look for BareMetalHost outside of their own one, to allow that, you need to execute this:

```sh
oc patch provisioning provisioning-configuration --type merge -p '{"spec":{"watchAllNamespaces": true}}'
```

Once done, we will need to create our new BareMetalHosts on the same Namespace as the Assisted Service is running in order to allow Bare Metal Agent Controller to decorate the BMH with the URL of the ISO generated by the Assisted Installer.

First, we will need to create a Secret that contains the credentials for the BMC (Baseboard Management Controller), for that we need a secret like this:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: bmc-secret1
  namespace: open-cluster-management
data:
  password: YWxrbm9wZmxlcgo=
  username: YW1vcmdhbnQK
type: Opaque
```

And these are the most important fields in the BareMetalHost CRD:

- `metadata.labels.infraenvs.agent-install.openshift.io`: This needs to point InfraEnv's name of the manifest created in the previous step.
- `metadata.annotations.inspect.metal3.io`: This one should be set to disabled.
- `spec.online`: Should be true, in order to allow Ironic to boot the node.
- `spec.automatedCleaningMode`: This should be `disabled`, this is only relevant on PXE environments.
- `spec.bootMACAddress`: This is the MAC address that will be used to boot the node.
- `spec.bmc.address`: Redfish address that you need to reach to contact with the node's BMC.
- `spec.bmc.credentialsName`: The secret's name that contains the access to the BMC.

So, with that let's take a look to a `BareMetalHost` CR sample

```yaml
apiVersion: metal3.io/v1alpha1
kind: BareMetalHost
metadata:
  name: lab-agent1
  namespace: open-cluster-management
  labels:
    infraenvs.agent-install.openshift.io: "lab-env"
  annotations:
    inspect.metal3.io: disabled
spec:
  online: true
  bmc:
    address: redfish-virtualmedia+http://[2620:52:0:1302::d7c]:8000/redfish/v1/Systems/3e6f03bb-2301-49c9-a562-ad488dca513c
    credentialsName: bmc-secret1
    disableCertificateVerification: true
  bootMACAddress: ee:bb:aa:ee:1e:1a
  automatedCleaningMode: disabled
```

#### Manual Spoke cluster deployment

This flow is easier but it's fully manual:

1. We need to get the ISO URL from the InfraEnv CR with this command:
   ```sh
   oc get infraenv lab-env -o jsonpath={.status.isoDownloadURL}
   ```
1. Then download and host it on a HTTPD server.
1. Access to the BMC and try to mount the ISO hosted
1. Boot the node and wait for it to be self-registered against the Assisted Service.
1. Now we need to check the AgentClusterInstall to verify that on the .status.conditions all the requirements are met.
   ```sh
   oc get agentclusterinstall -o yaml
   ```
1. Edit the agent(s) registered, changing the hostname and approving them.
1. The deployment will be self-triggered.

With this, the flow should finish completely, in case it doesn't, there might be some issues on the AgentClusterInstall CR created, so the best way to move forward is to examine the troubleshooting documentation.

## Wrap up

We hope that with this article we were able to achieve showcasing how Zero Touch Provisioning can be used for deploying OpenShift Clusters that fit the Telco 5G use case and how Red Hat Advanced Cluster can be leveraged
