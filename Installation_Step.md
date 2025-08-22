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
