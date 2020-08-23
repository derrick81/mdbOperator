# Setting up standalone MongoDB on local k3s cluster (quickly)
The key steps involved in getting this minimal setup up and running include:
1. Spinning up a k3s cluster
2. Installing the MongoDB Kubernetes Operator
3. Deploying Ops Manager (optional)
4. Deploying the standalone cluster

Note that this is meant as a quick (and perhaps opinionated) guide to have a minimal standalone setup. For detailed instructions on how to set things up according to your requirements or to use other methods such as helm, please refer to official documentation at https://docs.mongodb.com/kubernetes-operator/stable/.

## 1. Spinning up the k3s cluster using k3d
Details can be found at k3d guide at https://k3d.io/usage/guides/exposing_services/.
I created my cluster named mdb-cluster with 3 worker nodes (agents) with the entire NodePort range mapped in Docker containers:

`k3d cluster create mdb-cluster --agents 3 -p "30000-32767:30000-32767@server[0]"`

The reason for mapping the entire NodePort range is so that I do not have to figure what ports need to be exposed as a NodePort service prior to even creating any Kubernetes resource objects.

## 2. Installing the MongoDB Kubernetes Operator
When the k3s cluster has been created, the right context should have been set to your kubectl. Execute the list of commands here to get the MongoDB Operator setup. No modifications to the respective files are expected.
* Create mongodb namespace which is the namespace used throughout the rest of this guide\
`kubectl create namespace mongodb`
* Deploy MongoDB CRDs to the cluster\
`kubectl apply -f crds.yaml`
* Deploy the MongoDB Operator to the cluster\
`kubectl apply -f mongodb-enterprise.yaml`

## 3. Deploy Ops Manager (Optional)
The steps here will help you install the Ops Manager here directly to the cluster and operated in the most minimal way possible. Alternatively you can make use of existing Ops Manager or Cloud Manager.
* Point the kubectl context to work directly in mongodb namespace\
`kubectl config set-context $(kubectl config current-context) --namespace=mongodb`
* Create the secret object for Ops Manager credentials. Please fill in with your own password.\
`kubectl create secret generic adminusercredentials --from-literal=Username='admin' --from-literal=Password='<FILL_IN_PASSWORD>' --from-literal=FirstName='OpsMan' --from-literal=LastName='Administrator'`
* Create the Ops Manager\
`kubectl apply -f opsmgr.yaml`
* Use this to check for the status of the Ops Manager setup. It will take relatively longer than the other steps (~15 to 20 minutes)\
`kubectl get om -o yaml -w`
Once the Ops Manager deployment is completed in k8s, please log in with the Ops Manager credentials above and follow the wizard to complete the setup itself. You should also set up at least one organization and a corresponding project in preparation for the next step. Refer to the docs at https://docs.opsmanager.mongodb.com/current/tutorial/manage-organizations/ for details.

## 4. Deploy the standalone cluster
* Create the secret object for API access to Ops Manager. If you are not familiar with creating the API Key credentials for a particular org and project in Ops Manager, please refer to https://docs.opsmanager.mongodb.com/current/tutorial/manage-programmatic-api-keys/.\
`kubectl -n mongodb create secret generic operatorcreds --from-literal='user=<FILL_IN_API_PUBLIC_KEY>' --from-literal='publicApiKey=<FILL_IN_API_PRIVATE_KEY>'`
* Identify the Ops Manager instance, org and proj which the MongoDB process will be associated to.\
`kubectl create configmap projcm --from-literal="baseUrl=<FILL_IN_OPS_MGR_BASE_URL>" --from-literal="projectName=<FILL_IN_PROJECT_NAME>" --from-literal="orgId=<FILL_IN_24_CHARACTER_ORG_ID>`
* Create standalone MongoDB\
`k apply -f standalone.yaml`
* Use this to check for the status of the MongoDB setup.\
`k get mdb mdb-standalone -o yaml -w`
* Expose standalone MongoDB as NodePort service on port 30017. You may want to update this port number in the yaml file if it is already in use.\
`k apply -f standalone-svc.yaml`


