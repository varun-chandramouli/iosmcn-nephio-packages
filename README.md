# iosmcn-nephio-packages

After provisioning the cluster using the byoh script follow the following steps on the server or VM where you want to deploy the iosmcn-core

Step 1:
# Increase inotify limits 
sudo sysctl fs.inotify.max_user_watches=524288
sudo sysctl fs.inotify.max_user_instances=512
sudo sysctl fs.file-max=2097152
echo "fs.inotify.max_user_watches=524288" | sudo tee -a /etc/sysctl.conf
echo "fs.inotify.max_user_instances=512" | sudo tee -a /etc/sysctl.conf
echo "fs.file-max=2097152" | sudo tee -a /etc/sysctl.conf

Step 2: 
# Install the dependencies 
sudo apt update
sudo apt updgrade -y
sudo apt install make sshpass ansible -y

Step 3: 
Download the iosmcn core release  and run  make 5gc-router-install

Step 4:
Label the core cluster  with the following commands:
kubectl label nodes <core-node-name> node-role.aetherproject.org=omec-upf --context <core-cluster-context>
kubectl label nodes <core-node-name> node-role.aetherproject.org/omec-upf= --context <core-cluster-context>

Step 5:
Prepare the core-cluster with required package variants

5a. On the Server / VM where the nephio management cluster is deployed add this git repository

cat << EOF | kubectl apply -f - 
apiVersion: config.porch.kpt.dev/v1alpha1
kind: Repository
metadata:
  name: iosmcn-nephio-packages
  namespace: default
  labels:
    kpt.dev/repository-access: read-only
    kpt.dev/repository-content: external-blueprints
spec:
  content: Package
  deployment: false
  git:
    branch: main
    directory: /
    repo: https://github.com/varun-chandramouli/iosmcn-nephio-packages.git
  type: git


5b. Open nephio-webui  using the link :   http://<vm/server ip>:7007

5c. Click on mgmt link in the Repositories line of the  homepage. 

5d. Click on add deployment on top right corner.

5e. Under action select Create a new deployment by cloning a external blueprint.

5f. Under Source external blueprint repository select iosmcn-nephio-packages.

5g. Under external blueprint to clone select nephio-workload-cluster. Click on next

5h. In the metadata section change the name to core . Click on next.

5i. In the namespace section , click on next.

5j. In the validate section , click on next.

5k. In the confirm section , click on create deployment.

5l. Click on edit on the top right corner.

5m. Click on the last option of WorkloadCluster and click on show yaml view in the popup. 

5n. Change the masterInterface to the VM / server's data interface on which core cluster is deployed. Click on Save

5o. Click on save on top right corner.  in next screen click on propose on top right corner. In next screen click on approve. 

5p. This will install the required nephio packages for the deploying workloads and also the gitea repository for core cluster. 

5q. Once everything is ready we need to increase the MULTUS POD's memory  
  a) in the nephio homepage . Click on mgmt-staging link in the Repositories secion.
  b) select core-multus and in the next screen click on create new revision. In the next screen click on edit.
  c) Select DaemonSet and edit the memory value to 500Mi in both requests and limits section. Click on save.
  d) Click on save option in top right corner. Select propose and then approve.
  
5r.The rootsync installed in the core cluster  needs to be updated with correct IP of the core gitea repo.
 a) in the nephio homepage . Click on mgmt-staging link in the Repositories secion.
 b) select core-rootsync and in the next screen click on create new revision. In the next screen click on edit.
 c) select starlark Run option . Replace 172.x.x.x with the IP of the VM / Server where the gitea  is installed. this will usually be the same VM / server where the nephio manangement cluster is running. Click on save.
 d)  Click on save option in top right corner. Select propose and then approve.
 e) verify rootsync is created successfully in the core cluster with the command : 
    $ kubectl get rootsyncs.configsync.gke.io --context core-admin@core -A

5s. Deploy the iosmcncore-flux helm chart using the nephio web-ui.


After provisioning the cluster using the byoh script follow the following steps on the server or VM where you want to deploy the iosmcn-ran

Step 1:
# Check current state
cat /sys/kernel/mm/hugepages/hugepages-1048576kB/nr_hugepages

# Allocate 20 x 1Gi hugepages (leaves ~108Gi for everything else)
echo 20 | sudo tee /sys/kernel/mm/hugepages/hugepages-1048576kB/nr_hugepages

# Verify it took effect
cat /sys/kernel/mm/hugepages/hugepages-1048576kB/nr_hugepages
cat /sys/kernel/mm/hugepages/hugepages-1048576kB/free_hugepages


echo 'vm.nr_hugepages=20' | sudo tee /etc/sysctl.d/99-hugepages.conf
sudo sysctl -p /etc/sysctl.d/99-hugepages.conf


Step 2:
Repeat steps 5b to 5r , changing the name of cluster and reepo as ran . 

Step 3: 
Deploy the iosmcn-ran-cucp-flux , iosmcn-ran-cuup-flux and iosmcn-ran-du-flux helm charts in that order.
