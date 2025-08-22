# Step to Installation Storage Scale Container Native
1.  Pre-requisite on Storage Scale side please follow the [How-To-Storage-Scale](https://github.com/AhBestt/How-To-Storage-Scale/)
___
2.  Install OpenShift Cluster
___
3. Create users on Bastion Host <br>
  -  Create user <br>
`useradd -m -s /bin/bash -G wheel <username>` <br>
  - Create password for that user <br>
`passwd <username>`
  - Grant sudo privileges to that user <br>
`echo "core ALL=(ALL) ALL" >> /etc/sudoers`
___
4. Download kubeconfig from the openshift console and configure kubeconfig <br>
`sudo chown core:core kubeconfig`<br>
`mkdir .kube` <br>
`mv kubeconfig .kube/` <br>
`vi .bash_profile`
- add following <br>
```bash
export KUBECONFIG=$HOME/.kube/kubeconfig
```
- Relogin after configure
___
5. Configure Infra Nodes and Ingress Controller
- Verify Ingress <br>
`oc get pod -A -o wide | grep ingress` <br>
- Labels Infra node <br>
`oc label node <FQDN_Infra_Node1_name> node-role.kubernetes.io/infra=true` <br>
`oc label node <FQDN_Infra_Node2_name> node-role.kubernetes.io/infra=true` <br>
- Check Node Taints <br>
```bash
oc get node
oc describe node <FQDN_Infra_Node1_name> | grep -i taints
oc describe node <FQDN_Infra_Node2_name> | grep -i taints
oc describe node <FQDN_Master_Node1_name> | grep -i taints
```
- Add Taints to Infra Nodes <br>
`oc adm taint node <FQDN_Infra_Node1_name> node-role.kubernetes.io/infra:NoSchedule` <br>
`oc adm taint node <FQDN_Infra_Node2_name> node-role.kubernetes.io/infra:NoSchedule` <br>
- Verify Taints <br>
`oc describe node <FQDN_Infra_Node1_name> | grep -i taints` <br>
`oc describe node <FQDN_Infra_Node2_name> | grep -i taints` <br>
- Edit Ingress Controller
`oc edit ingresscontroller default -n openshift-ingress-operator`<br>
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
  `oc get pod -n openshift-ingress -o wide` <br>
  `oc get co | grep ingress` # should show True <br>
- Create Infra MachineConfigPool <br>
  -  Create the `infra_mcp.yaml` file [infra_mcp.yaml](https://github.com/AhBestt/How-To-Storage-Scale-Container-Native/blob/main/infra_yaml/infra_mcp.yaml) <br>
  -  Apply the MCP <br>
  `oc apply -f infra_mcp.yaml` <br>
  -  Verify MCP status <br>
  `oc get mc` # Should show render-infra
  `oc get mcp` # Should show infra UP
___
6. Download operator on openshift-console from OperatorHub <br>
   - To show OpenShift Console run: <br>
   `oc whoami --show-console` <br>
   - Download Kubernetes NMState Operator and create NMState as default <br>
   - Setting IP as needed [bond2.yaml](https://github.com/AhBestt/How-To-Storage-Scale-Container-Native/blob/main/infra_yaml/bond2.yaml) <br>
   
