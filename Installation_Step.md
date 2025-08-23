# Step to Installation Storage Scale Container Native
1.  Pre-requisite on Storage Scale side please follow the [How-To-Storage-Scale](https://github.com/AhBestt/How-To-Storage-Scale/)
___
2.  Install OpenShift Cluster
___
3. Create users on Bastion Host <br>
  -  Create user <br>
  ```bash
  useradd -m -s /bin/bash -G wheel <username>`
  ``` 
  - Create password for that user <br>
  ```bash
  passwd <username>
  ```
  - Grant sudo privileges to that user <br>
  ```bash
  echo "core ALL=(ALL) ALL" >> /etc/sudoers
  ```
___
4. Download kubeconfig from the openshift console and configure kubeconfig <br>
  ```bash
  sudo chown core:core kubeconfig
  mkdir .kube
  mv kubeconfig .kube/
  vi .bash_profile
  ```
- Add following 
```bash
export KUBECONFIG=$HOME/.kube/kubeconfig
```
- Relogin after configure
___
5. Configure Infra Nodes and Ingress Controller
- Verify Ingress <br>
  ```bash
  oc get pod -A -o wide | grep ingress
  ```
- Labels Infra node <br>
  ```bash
  oc label node <FQDN_Infra_Node1_name> node-role.kubernetes.io/infra=true
  oc label node <FQDN_Infra_Node2_name> node-role.kubernetes.io/infra=true
  ```
- Check Node Taints <br>
  ```bash
  oc get node
  oc describe node <FQDN_Infra_Node1_name> | grep -i taints
  oc describe node <FQDN_Infra_Node2_name> | grep -i taints
  oc describe node <FQDN_Master_Node1_name> | grep -i taints
  ```
- Add Taints to Infra Nodes <br>
  ```bash
  oc adm taint node <FQDN_Infra_Node1_name> node-role.kubernetes.io/infra:NoSchedule
  oc adm taint node <FQDN_Infra_Node2_name> node-role.kubernetes.io/infra:NoSchedule
  ```
- Verify Taints <br>
  ```bash
  oc describe node <FQDN_Infra_Node1_name> | grep -i taints
  oc describe node <FQDN_Infra_Node2_name> | grep -i taints
  ```
- Edit Ingress Controller <br>
  ```bash
  oc edit ingresscontroller default -n openshift-ingress-operator
  ```
  - Add the following configuration under `spec: ` <br>
  ```bash
  spec:
    nodePlacement:
      nodeSelector:
        matchLabels:
          node-role.kubernetes.io/infra: "true"
      tolerations:
      - effect: NoSchedule
        key: node-role.kubernetes.io/infra
        operator: Exists
  ```
- Verify Ingress Pods <br>
  ```bash
  oc get pod -n openshift-ingress -o wide
  oc get co | grep ingress
  ```
  > Expected result: Should show True 
- Create Infra MachineConfigPool <br>
  -  Create the `infra_mcp.yaml` file [infra_mcp.yaml](https://github.com/AhBestt/How-To-Storage-Scale-Container-Native/blob/main/infra_yaml/infra_mcp.yaml) <br>
  -  Apply the MCP <br>
    ```bash
    oc apply -f infra_mcp.yaml
    ```
  -  Verify MCP status <br>
    ```bash
    oc get mc 
    oc get mcp
    ```
  > Expected result: Render-infra is shown from mc and Infra is shown from mcp 
___
6. Download operator on openshift-console from OperatorHub <br>
- To show OpenShift Console run: <br>
  ```bash
  oc whoami --show-console
  ```
- Download Kubernetes NMState Operator and create NMState as default <br>
- Setting IP as needed for Worker Node by [bond2.yaml](https://github.com/AhBestt/How-To-Storage-Scale-Container-Native/blob/main/infra_yaml/bond2.yaml) and [vlan.yaml](https://github.com/AhBestt/How-To-Storage-Scale-Container-Native/tree/main/infra_yaml)<br>
- After configuration run the yaml file <br>
  ```bash
  oc apply -f bond2.yaml
  oc apply -f vlan.yaml
  ```
___
7. Install Storage Scale Container Native <br>
- Unlabel worker on Infra Node <br>
  ```bash
  oc label node/<FQDN_Infra_Node1_Name> node-role.kubernetes.io/worker-
  oc label node/<FQDN_Infra_Node2_Name> node-role.kubernetes.io/worker-
  ```
- Download the Worker Node Configuration YAML from [IBM Docs](https://www.ibm.com/docs/en/scalecontainernative/5.2.3?topic=premise-worker-node-configuration) matching the deployed version (e.g., 5.2.3) and the machine architecture (e.g., x86_64) <br>
  - Apply yaml file <br>
    ```bash
    oc apply -f mco_x86_64.yaml
    ```
  - Check Status for mcp and wait until the update completes <br>
    ```bash
    oc get mcp
    ```
- Verify Worker Node and Label <br>
  - Verify Worker Node (Ensure the worker label is applied only to Worker Nodes) <br>
    ```bash
    oc get node -l node-role.kubernetes.io/worker
    ```
  - label Worker Node <br>
    ```bash
    oc label nodes -l node-role.kubernetes.io/worker= scale.spectrum.ibm.com/daemon-selector=
    ```
- Validate kernel package and secure boot <br>
  - Validate kernel package is installed <br>
    ```bash
    oc get nodes -lscale.spectrum.ibm.com/daemon-selector= -ojsonpath="{range .items[*]}{.metadata.name}{'\n'}" | xargs -I{} oc debug node/{} -T -- chroot /host sh -c "rpm -q kernel-devel"
    ```
  - Validate secure boot is disabled <br>
    ```bash
    oc get nodes -lscale.spectrum.ibm.com/daemon-selector= -ojsonpath="{range .items[*]}{.metadata.name}{'\n'}" | xargs -I{} oc debug node/{} -T -- chroot /host sh -c "mokutil --sb-state"
    ```
- Install the operator into the target environment <br>
  ```bash
  oc apply -f https://raw.githubusercontent.com/IBM/ibm-spectrum-scale-container-native/v5.2.3.x/generated/scale/install.yaml
  ```
  - Validate Namespace <br>
  ```bash
  oc get namespaces | grep ibm-spectrum-scale
  ```
  >Expected result: The following namespaces appear and in Ready status <br>
  > ibm-spectrum-scale <br>
  > ibm-spectrum-scale-csi <br>
  > ibm-spectrum-scale-dns <br>
  > ibm-spectrum-scale-operator <br>
  
  - Validate pod <br>
  ```bash
  oc get pods -n ibm-spectrum-scale-operator
  ```
  >Expected result: The following pod appear and in Ready status <br>
  >ibm-spectrum-scale-controller-manager-<generated_number> <br>
  ```bash  
  oc get pods -n ibm-spectrum-scale-csi
  ```
  > Expected result: The following pod appear and in Ready status <br>
  > ibm-spectrum-scale-csi-operator-<generated_number> <br>
- Export Key and Create name space pull secret <br>
  -  Export Key Entitlement <br>
    ```bash
     export ENTITLEMENT_KEY=<REPLACE WITH ICR ENTITLEMENT KEY>
    ```
  - Create a docker-registry secret for each namespace <br>
    ```bash
     for namespace in ibm-spectrum-scale ibm-spectrum-scale-operator ibm-spectrum-scale-dns ibm-spectrum-scale-csi; do
     kubectl create secret docker-registry ibm-entitlement-key -n ${namespace} \
     --docker-server=cp.icr.io \
     --docker-username=cp \
     --docker-password=${ENTITLEMENT_KEY}
     done
    ```
  - Unset the export <br>
    ```bash
    unset ENTITLEMENT_KEY
    ```
- Download Cluster CR sample from [IBM Doc](https://www.ibm.com/docs/en/scalecontainernative/5.2.3?topic=resources-cluster)<br>
  - For Red Hat OpenShift Cluster CR <br>
  ```bash
  curl -fs https://raw.githubusercontent.com/IBM/ibm-spectrum-scale-container-native/v5.2.3.x/generated/scale/cr/cluster/cluster.yaml > cluster.yaml || echo "Failed to download Cluster sample CR"
  ```
  - Configure and apply cluster <br>
  ```bash
  oc apply -f cluster.yaml
  ```
  -  Verify that the Operator has created by checking the pods <br>
  ```bash
  oc get pod -n ibm-spectrum-scale
  ```
    - A sample output is shown: <br>
    ```bash 
    NAME                               READY   STATUS    RESTARTS   AGE 
    ibm-spectrum-scale-gui-0           4/4     Running   0          5m45s 
    ibm-spectrum-scale-gui-1           4/4     Running   0          2m9s 
    ibm-spectrum-scale-pmcollector-0   2/2     Running   0          5m15s 
    ibm-spectrum-scale-pmcollector-1   2/2     Running   0          4m11s 
    worker0                            2/2     Running   0          5m43s 
    worker1                            2/2     Running   0          5m43s 
    worker3                            2/2     Running   0          5m45s 
    ```
- Create RemoteCluster <br>
  - Create secret for storage cluster GUI users <br>
  ```bash
  oc create secret generic cnsa-remote-mount-storage-cluster-1 --from-literal=username='cnsa_storage_gui_user' \
  --from-literal=password='cnsa_storage_gui_password' -n ibm-spectrum-scale
  ```
  - Label the secret <br>
  ```bash
  oc label secret cnsa-remote-mount-storage-cluster-1 -n ibm-spectrum-scale product=ibm-spectrum-scale
  ```
  - Create Configmap follow [IBM Docs](https://www.ibm.com/docs/en/scalecontainernative/5.2.3?topic=remotecluster-configuring-certificate-authority-ca-certificates) <br>
  > If CSI requested, please create configmap on namespaces `ibm-spectrum-scale-csi` too <br>
  - Download a copy of the sample `remotecluster.yaml` <br>
  ```bash
  curl -fs https://raw.githubusercontent.com/IBM/ibm-spectrum-scale-container-native/v5.2.3.x/generated/scale/cr/remotecluster/remotecluster.yaml > remotecluster.yaml || echo "Failed to download RemoteCluster sample CR"
  ```
  - Configure remote cluster and apply <br>
  ```bash
  oc apply -f remotecluster.yaml
  ```
  - Validate status of remotecluster <br>
  ```bash
  oc get remotecluster -n ibm-spectrum-scale
  ```
    - A sample output is shown <br>
    ```bash
    NAME                   READY   AGE
    remotecluster-sample   True    25h
    ```
  - Download a copy of the sample `filesystem.remote.yaml` <br>
  ```bash
  curl -fs https://raw.githubusercontent.com/IBM/ibm-spectrum-scale-container-native/v5.2.3.x/generated/scale/cr/filesystem/filesystem.remote.yaml > filesystem.remote.yaml || echo "Failed to download Filesystem sample CR"
  ```
  - Configure and apply filesystem <br>
  ```bash
  oc apply -f filesystem.remote.yaml
  ```
  - Validate status <br>
  ```bash
  oc get fs -n ibm-spectrum-scale
  ```
    - A sample output is shown <br>
    ```bash
    NAME            ESTABLISHED   AGE
    remote-sample   True          25h
    ```
___
8.Test Creating Pod <br>
  - Check Storage Class <br>
  ```bash
  oc get sc
  ```
  -  Test create pvc, download a [pvc.yaml](https://github.com/AhBestt/How-To-Storage-Scale-Container-Native/blob/main/infra_yaml/pvc.yaml) and apply the yaml file <br>
  ```bash
  oc apply -f pvc.ymal
  ```
  - Check PVC status is bonding <br>
  ```bash
  oc get pvc
  ```
  - Test create pod, download a [pod.yaml](https://github.com/AhBestt/How-To-Storage-Scale-Container-Native/blob/main/infra_yaml/pod.yaml) and apply the yaml file <br>
  ```bash
  oc apply -f pod.yaml
  ```
  - Validate Pod is running <br>
  ```bash
  oc get pod
  ```
  > Expected result: Pod is running
