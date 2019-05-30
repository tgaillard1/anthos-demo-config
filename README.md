# anthos-demo-config

******************************************************************
******************************************************************
*******   Anthos demo -- Quick Start   *******
*******   Reference Implementation   *******
*******   Main document --> go/anthos-workshop-reference   *******
******************************************************************
******************************************************************
Enable Access (Start with this step -- can take 2 days to process)
******************************************************************

REQUEST ACCESS for gcct-demos folder 
https://sphinx.corp.google.com/sphinx/

EXEMPT your project from GCE Enforced
http://cloud-exemptions.googleplex.com

         ################### GKE OnPrem Whitelist ###################
         (project, user, service account)
         gcloud iam service-accounts create anthos-connect
         Fill out the form with
         anthos-connect@[YOUR_PROJECT].iam.gserviceaccount.com
         #######################################################

******************************************************
Enable API's
******************************************************
gcloud config set project YOUR_PROJECT

gcloud services enable \
    cloudresourcemanager.googleapis.com \
    container.googleapis.com \
    gkeconnect.googleapis.com \
    gkehub.googleapis.com \
    serviceusage.googleapis.com \
    containerregistry.googleapis.com \
    cloudbuild.googleapis.com \
    sourcerepo.googleapis.com \
    run.googleapis.com

******************************************************
Clone Source Repo & Set the base directory and paths
******************************************************

git clone https://github.com/GoogleCloudPlatform/anthos-workshop.git anthos-demo

cd anthos-demo

Set the base directory and paths
source ./env

******************************************************
Execute the scripts
******************************************************

./bootstrap-workshop.sh

This script does the following:
Downloads client-side tools: kubectx, Helm, istioctl
Creates a GKE (on GCP) cluster named central 
Creates a second, non-GKE Kubernetes cluster in GCP (Kops / GCE) 
Installs the Anthos Config Management operator on both Kubernetes clusters 
Installs open-source Istio on both Kubernetes clusters 

        ################### Firewall issues ###################
        Run this to see if the firewall rul is in place:

        gcloud compute firewall-rules list --format yaml | \
        grep https-api-remote-k8s-local

        Output:
        name: https-api-remote-k8s-local

        If not in place or your shell has been restarted with new IP run this command -->
        ./common/remote-k8s-access-fw.sh
        #######################################################

******************************************************
Register Clusters with Hub (Console) 
******************************************************

******* Validate cluster status
kubectx remote
kubectl get nodes

******* Add local repository *******
sudo gcloud components repositories add \
https://storage.googleapis.com/gkehub-gcloud-dist-beta/components-2.json

sudo gcloud components update --quiet

******* Create GCP Service Account for cluster registration *******
export PROJECT=$(gcloud config get-value project)
export GKE_CONNECT_SA=anthos-connect
export GKE_SA_CREDS=$WORK_DIR/$GKE_CONNECT_SA-creds.json

# Create and Update bindings for Service Account
# Assign it GKE Connect rights

gcloud projects add-iam-policy-binding $PROJECT \
--member="serviceAccount:$GKE_CONNECT_SA@$PROJECT.iam.gserviceaccount.com" \
--role="roles/gkehub.connect"

# Create and download a key
gcloud iam service-accounts keys create $GKE_SA_CREDS \
  --iam-account=$GKE_CONNECT_SA@$PROJECT.iam.gserviceaccount.com \
  --project=$PROJECT


******* Use Console to Register with Hub *******

cat $GKE_SA_CREDS

Go to --> GCP --> GKE --> Clusters --> Register Cluster
Enter --> Cluster name = remote
Paste SA 

In the ensuing screen, click DOWNLOAD GKE CONNECT MANIFEST and save the file.

******* Create connect file *******
vi $WORK_DIR/remote-cluster-gke-connect-config.yaml

Open MANIFEST doc locally and copy the entire contents into the remote-cluster-gke-connect-config.yaml created above -- save firewall

******* Then run this to connect to cluster *******
export REMOTE_CLUSTER_NAME_BASE="remote"

kubectx $REMOTE_CLUSTER_NAME_BASE
kubectl apply -f $WORK_DIR/remote-cluster-gke-connect-config.yaml

If successful you get --> GKE Connect agent for cluser remote is connected -- in Console

Apply any desired labels and click Register

******* Login to cluser -- Note: Registered cluster with SA, not local IAM user; tgaillard *******
******* Create Token *******

export KSA=remote-admin-sa
kubectl create serviceaccount $KSA
kubectl create clusterrolebinding ksa-admin-binding \
    --clusterrole cluster-admin \
    --serviceaccount default:$KSA

# Retrieve Token for login process
printf "\n$(kubectl describe secret $KSA | sed -ne 's/^token: *//p')\n\n"

Go to console and select token and paste in the results from the above command

******************************************************
Deploy Apps
******************************************************
kubectx central
kubectl run nginx-central --image=nginx --replicas=3

kubectx remote
kubectl run nginx-remote --image=nginx --replicas=3

kubectl config use-context checkout
kubectl run nginx-check --image=nginx --replicas=3 --use-context=checkout

******************************************************
Central Policy Management
******************************************************

******* Configure Git *******
 
Fork the sample repo to your git location so you can push changes to your own copy --> https://github.com/cgrant/config-repos

cd $HOME
git clone https://github.com/tgaillard1/config-repo.git
cd $HOME/config-repo

export GITHUB_ACCOUNT=tgaillard1
export REPO_URL=https://github.com/tgaillard1/config-repo.git



                ******* Google Cloud Source Repositories (GCSR) *******

                export GCLOUD_ACCOUNT=$(gcloud config get-value account)
                export REPO_URL=https://source.developers.google.com/p/${PROJECT}/r/config-repo

                git remote remove origin
                git config credential.helper gcloud.sh
                git remote add origin $REPO_URL

                gcloud source repos create config-repo
                git push -u origin master

                ******* GCSR generate ssh keypair *******

                export NOMOS_SSH_KEY=id_rsa.nomos
                ssh-keygen -t rsa -b 4096 \
                -C "$GCLOUD_ACCOUNT" \
                -N '' \
                -f $HOME/.ssh/$NOMOS_SSH_KEY

                ******* Pair to clusters *******

                kubectx central
                kubectl create secret generic git-creds \
                --namespace=config-management-system \
                --from-file=ssh=$HOME/.ssh/$NOMOS_SSH_KEY

                kubectx remote
                kubectl create secret generic git-creds \
                --namespace=config-management-system \
                --from-file=ssh=$HOME/.ssh/$NOMOS_SSH_KEY

                ******* Add Public key to GCSR *******

                Navigate to: https://source.cloud.google.com/user/ssh_keys
                Select “Register SSH key”
                Key Name = anthos demo key
                Key* =
                Paste contents from --> cat $HOME/.ssh/$NOMOS_SSH_KEY.pub

******* Validate structure and Connect *******

tree . (results should show istio-system and logging namespaces)
echo $REPO_URL
cat $BASE_DIR/config-management/config_sync.yaml (verify it has right variables)

@@@@@@@@@@@@@ Split screens
Right screen --

export REMOTE=remote
export CENTRAL=central

watch \
    "echo '## $CENTRAL ##\nNamespaces'; \
    kubectl --context $CENTRAL get ns; \
    echo '\nLogging Pods'; \
    kubectl --context $CENTRAL get po -n logging; \
    echo '\n## $REMOTE ##\nNamespaces'; \
    kubectl --context $REMOTE get ns; \
    echo '\nLogging Pods'; \
    kubectl --context $REMOTE get po -n logging"

@@@@@@@@@@@@@ 
Left screen

--- Git Hub ---
export REMOTE=remote
export CENTRAL=central

kubectx $REMOTE

# Replace variables and stream results to kubectl apply
cat $BASE_DIR/config-management/config_sync.yaml | \
  sed 's@<REPO_URL>@'"$REPO_URL"'@g' | \
  sed 's@<CLUSTER_NAME>@'"$REMOTE"'@g' | \
  kubectl apply -f -

kubectx $CENTRAL

# Replace variables and stream results to kubectl apply
cat $BASE_DIR/config-management/config_sync.yaml | \
  sed 's@<REPO_URL>@'"$REPO_URL"'@g' | \
  sed 's@<CLUSTER_NAME>@'"$CENTRAL"'@g' | \
  kubectl apply -f -

In a few moments you should see the logging namespace and fluentd daemonset automatically applied to all the clusters (central and remote).

-----
******* Push a config update *******

----- namespace

mkdir namespaces/checkout

cat <<EOF > namespaces/checkout/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: checkout
EOF

tree .


git config --global user.email timlgaillard@gmail.com
git config --global user.name tgaillard1
git add . && git commit -m 'adding checkout namespace'
git push origin master

----- Quoata 
cat <<EOF > namespaces/checkout/quota.yaml
kind: ResourceQuota
apiVersion: v1
metadata:
  name: quota
spec:
  hard:
    pods: "3"
    cpu: "1"
    memory: 1Gi
EOF

tree .

git config --global user.email timlgaillard@gmail.com
git config --global user.name tgaillard1
git add . && git commit -m 'adding quoata to checkout namespace'
git push origin master


remove
git config --global user.email timlgaillard@gmail.com
git config --global user.name tgaillard1
git add . && git commit -m 'remove checkout namespace'
git push origin master


******************************************************
******************************************************
Istio Config 
******************************************************
******************************************************

******* Create a ConfigMap for the stub domain by running the following command.
cd $BASE_DIR
./hybrid-multicluster/istio-dns.sh

******* Run the following command to verify the stub domain created in the kube-dns configmap.
kubectl --context central -n kube-system get configmap kube-dns -o json | jq '.data'

kubectl --context central -n istio-system get configmap coredns -o json | jq -r '.data.Corefile'

******************************************************
******************************************************
*******     Deploy MicroService Application    *******
******************************************************
******************************************************
gcloud config set project gaillard-gcp
cd /home/tgaillard/anthos-demo/
source ./env

./common/remote-k8s-access-fw.sh

https://github.com/GoogleCloudPlatform/microservices-demo

@@@@@@@@@@@@@ Split screens -- cntl b shift 5
Right screen --

export REMOTE=remote
export CENTRAL=central

watch \
    "echo '## $CENTRAL ##\Pods'; \
    kubectl --context $CENTRAL get pods --namespace hipster2; \
    echo '\n## $REMOTE ##\Pods'; \
    kubectl --context $REMOTE get pods --namespace hipster1;"

@@@@@@@@@@@@@ 
Left screen  cntl b o (or arrow key)

-------- Deploy Applications

cd $BASE_DIR
./hybrid-multicluster/istio-deploy-hipster.sh

kubectl --context central -n hipster2 get all
kubectl --context remote -n hipster1 get all

******** Get External Service IP for Access to App ********
kubectl --context central get -n istio-system service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

******** Demonstrate replication controler ********
kubectl --context remote -n hipster1 delete pod ADSERVICE-SERVICE

********  Migrate Applications to Central Cluster  ********
cd $BASE_DIR
./hybrid-multicluster/istio-migrate-hipster.sh

******************************************************
*******  Demos Anthos Configuration Management *******
******************************************************

******* Validate structure and Connect *******

cd /home/tgaillard/config-repo
tree . (results should show istio-system and logging namespaces) --> https://github.com/tgaillard1/config-repo

@@@@@@@@@@@@@ Split screens
Right screen --

export REMOTE=remote
export CENTRAL=central

watch \
    "echo '## Central Cluster ##\nNamespaces'; \
    kubectl --context $CENTRAL get ns; \
    echo '\nCentral Cluster Checkout Quota'; \
    kubectl --context $CENTRAL describe resourcequota -n checkout; \
    echo '\n## Remote Cluster ##\nNamespaces'; \
    kubectl --context $REMOTE get ns; \
    echo '\nRemote Cluster Checkout Quota'; \
    kubectl --context $REMOTE describe resourcequota -n checkout"


@@@@@@@@@@@@@ 
Left screen

******* Push a config update *******

cd /home/tgaillard/config-repo

tree .

mkdir /home/tgaillard/config-repo/namespaces/checkout
cat <<EOF > /home/tgaillard/config-repo/namespaces/checkout/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: checkout
EOF

tree .

git config --global user.email timlgaillard@gmail.com
git config --global user.name tgaillard1
git add . && git commit -m 'adding checkout namespace'
git push origin master

----- Quoata 
cat <<EOF > /home/tgaillard/config-repo/namespaces/checkout/compute-resources.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
spec:
  hard:
    pods: "4"
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
    requests.nvidia.com/gpu: 4
EOF

tree .

git config --global user.email timlgaillard@gmail.com
git config --global user.name tgaillard1
git add . && git commit -m 'adding quoata to checkout namespace'
git push origin master

----- remove locally chechout quota and namespace

kubectl --context $REMOTE delete resourcequota compute-resources -n checkout
kubectl --context $REMOTE delete ns checkout

----- remove chechout quota

rm /home/tgaillard/config-repo/namespaces/checkout/compute-resources.yaml

git add . && git commit -m 'remove checkout quotas'
git push origin master

----- remove chechout namespace

kubectl --context $REMOTE delete ns checkout

rm -rf /home/tgaillard/config-repo/namespaces/checkout/ 

git add . && git commit -m 'remove checkout namespace'
git push origin master


******************************************************
******************************************************
Remove applications and cluster
******************************************************
******************************************************

*******     Remove MicroService Application    *******

kubectl --context remote delete ns hipster1
kubectl --context central delete ns hipster2

*******     Remove cluster    *******
run --> cleanup-workshop.sh




