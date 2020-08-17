# Setting up standalone MongoDB on local k3s cluster (quickly)
The key steps involved in getting this minimal setup up and running include:
1. Spinning up a k3s cluster
2. Installing the MongoDB Kubernetes Operator
3. Deploying Ops Manager (optional)
4. Deploying the standalone cluster

Note that this is meant as a quick guide to have a minimal standalone setup. For detailed instructions on how to set things up according to your spec, please refer to official documentation at https://docs.mongodb.com/kubernetes-operator/stable/.

## 1. Spinning up the k3s cluster using k3d
Details can be found at k3d guide at https://k3d.io/usage/guides/exposing_services/.
I created my cluster named mdb-cluster with 3 worker nodes (agents) with the entire NodePort range mapped in Docker containers:
`k3d cluster create mdb-cluster --agents 3 -p "30000-32767:30000-32767@server[0]"`

## 2. Installing the MongoDB Kubernetes Operator
When the k3s cluster has been created, the right context should have been set to your kubectl. Execute the list of commands here to get the MongoDB Operator setup. No modifications to the respective files are expected.
* Create mongodb namespace
`k create namespace mongodb`
* Apply CRDs to the cluster
`k apply -f crds.yaml`
* Apply the Operator to the cluster
`k apply -f mongodb-enterprise.yaml`

## 3. Deploy Ops Manager (Optional)
The steps here will help you install the Ops Manager here directly to the cluster. Alternatively you can make use of existing Ops Manager or Cloud Manager.
* Point the kubectl context to work directly in mongodb namespace
`k config set-context $(kubectl config current-context) --namespace=mongodb`
* Create the secret object for Ops Manager. Please fill in with your own password.
`k create secret generic adminusercredentials --from-literal=Username='admin' --from-literal=Password='<FILL_IN_PASSWORD>' --from-literal=FirstName='OpsMan' --from-literal=LastName='Administrator'`
* Create the Ops Manager
`k apply -f opsmgr.yaml`
* Use this to check for the status of the Ops Manager setup. It will take relatively longer than the other steps (~15 to 20 minutes)
`kubectl get om -o yaml -w`

## 4. Deploy the standalone cluster
* Create the secret object for API access to Ops Manager. If you are not familiar with creating the API Key credentials for a particular org and project in Ops Manager, please refer to https://docs.opsmanager.mongodb.com/current/tutorial/manage-programmatic-api-keys/.
`k -n mongodb create secret generic operatorcreds --from-literal='user=<FILL_IN_API_PUBLIC_KEY>' --from-literal='publicApiKey=<FILL_IN_API_PRIVATE_KEY>'`
* Identify the Ops Manager instance, org and proj which the MongoDB process will be associated to.
`k create configmap projcm --from-literal="baseUrl=<FILL_IN_OPS_MGR_BASE_URL>" --from-literal="projectName=<FILL_IN_PROJECT_NAME>" --from-literal="orgId=<FILL_IN_24_CHARACTER_ORG_ID>`
* Create standalone MongoDB
`k apply -f standalone.yaml`
* Use this to check for the status of the MongoDB setup.
`k get mdb mdb-standalone -o yaml -w`
* Expose standalone MongoDB as NodePort service on port 30017. You may want to update this port number in the yaml file if it is already in use.
`k apply -f standalone-svc.yaml`


