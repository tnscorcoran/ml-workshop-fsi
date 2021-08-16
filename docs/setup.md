# Install the workshop tools
## Prerequisites
You'll need
- Helm, the Kubernetes package manager. It's available [here](https://helm.sh/docs/intro/install/)
- an OpenShift cluster - with admin rights. You can create one by following the instructions [here](http:/try.openshift.com)
- the OpenShift command line interface, _oc_ available [here](https://docs.openshift.com/container-platform/4.6/cli_reference/openshift_cli/getting-started-cli.html)

## Installation procedure

If you are running this as a workshop, it is recommended you fork this repo as there are changes you can make to your instance of the repo, that will simplify the experience for the students. See section _Updating Tool URLs_ below.

Do the following:
- Clone this repo (or a fork thereof if you are a facilitator for students) and change directory into the root dir, _ml-workshop_.  Create a variable *REPO_HOME*_ for this directory
```
git clone https://github.com/masoodfaisal/ml-workshop
cd ml-workshop
export REPO_HOME=`pwd`
```
- On the OpenShift console, choose the _Copy Login Command_ as shown and paste the _oc login ..._ command it gives to a terminal.
![](https://github.com/masoodfaisal/ml-workshop/blob/main/docs/images/29-copy-login-command.png)

- Create a new project on the terminal
```
oc new-project ml-workshop  
```

- On GUI, select click project ml-workshop to select it
![](https://github.com/masoodfaisal/ml-workshop/blob/main/docs/images/30-select-ml-workshop-project.png)

- Before installation, you may need to get your OpenShift cluster administrator to adjust your limit ranges - or delete if this a test cluster without resource pressures. This is because, there are some moderate resource requirements associated with this workshop, e.g. Jenkins alone requires 4 vCPU and 4 Gi memory and there are other resource hungry elements as well. These are set here:
![](https://github.com/masoodfaisal/ml-workshop/blob/main/docs/images/29-resource-limits.png)

- Next, install Open Data Hub Operator on the Operator Hub screen. Filter on _Open Data Hub_ and go with all the defaults. It will install in the opershift-operators namespace (this takes several minutes)
![](https://github.com/masoodfaisal/ml-workshop/blob/main/docs/images/1-2-operatorhub-odh.png)


At this point, on GUI go to Installed Operators and wait until the _Open Data Hub_ related operator is installed.
![](https://github.com/masoodfaisal/ml-workshop/blob/main/docs/images/1-5-operatorhub-install-succeededXXXXXXXXXX.png)


Now install the tools themselves:

- Open Data Hub tools including Jupyter, our authoring tool today
```
cd $REPO_HOME/helm/odh
helm install ml-odh .
```

- Install Jenkins Operator on the Operator Hub screen. Filter on _Jenkins_ and click the Jenkins box. 
![](https://github.com/masoodfaisal/ml-workshop/blob/main/docs/images/1-1-operatorhub-jen-1.png)


From here, go with all the defaults, Clicking Install and again click on Install on the next screen, ensuring Installed Namespace is ml-workshop. (this takes several minutes)
![](https://github.com/masoodfaisal/ml-workshop/blob/main/docs/images/1-1-operatorhub-jen-2.png)
![](https://github.com/masoodfaisal/ml-workshop/blob/main/docs/images/1-1-operatorhub-jen-3.png)



After a few minutes, on GUI go back to _Installed Operators_ and wait until these Open Data Hub and Jenkins operators are installed as shown: 
![](https://github.com/masoodfaisal/ml-workshop/blob/main/docs/images/1-5-operatorhub-install-succeeded-incl--spark-seldon.png)

If you encounter this, you'll need to delete an operator group.
![](https://github.com/masoodfaisal/ml-workshop/blob/main/docs/images/1-5-operatorhub-seldon-spark-dont-deploy.png)

**NOTE - Don't do this unless you have these 2 in a perpetual _UpgradePending_ state.** To get your operator groups, execute the following:
```
oc get og -n ml-workshop
```
You'll see something like the following:
![](https://github.com/masoodfaisal/ml-workshop/blob/main/docs/images/1-5-two-operator-groups.png)

Delete 2 operator groups by executing the following (substituting with your actual _ml-workshop-xxx operator group_)
```
oc delete og opendatahub
oc delete og ml-workshop-xxxx
```
The _opendatahub_ one will reinstantiate itself. Then after a minute or two, you should see the 5 operators installed on the _Installed Operators_ screen - as seen in the screenshot above.

- Jenkins
```
cd $REPO_HOME/helm/jenkins
helm install ml-jenkins .
```


- Now deploy _Minio_ - our object storage implemenation
```
cd $REPO_HOME/helm/minio
helm install ml-minio .
```

- Verta.ai - our model repository
```
cd $REPO_HOME/helm/modeldb
helm install ml-modeldb .
```

- Run a Kubernetes job to push data to Kafka - from where the data engineer will pull data. Prior to running this, go to your _Pods_ view (with namespace ml-workshop selected) and enure there are 3 Kafka pods and 3 zookeeper pods running. If not, wait until they are. Then execute this:
```
cd $REPO_HOME/helm/kafka-populator
helm install ml-kafka-populator .
```



Now it's time to test each of the tools installed. Each of the tools we use should have an OpenShift Route created, apart from Verta.ai, the model repository tool. So go ahead and create one for that:
```
oc expose svc ml-modeldb-webapp
```
## Test Routes
Then on the GUI, open the menu item _Networking->Routes_ and you'll see these routes:

![](https://github.com/masoodfaisal/ml-workshop/blob/main/docs/images/32-routes.png)

Test each route as follows:

- jenkins-ml-jenkins: login with your OpenShift credentials
- jupyterhub: 
The first thing we need to do, before we login, is install a custom Jupyter image that contains required libraries for the three data-science focused workshops. Then we label it so it appears in the Jupyter Spawn Image dropdown. For more on that, see [Adding custom notebook images](https://opendatahub.io/docs/administration/installation-customization/add-custom-image.html)

Run the following
```
oc project ml-workshop
cd $REPO_HOME/notebook-image
oc import-image ml-workshop-elyra --from='quay.io/ml-aml-workshop/elyra:0.0.1' --reference-policy=local --confirm
oc label is ml-workshop-elyra 'opendatahub.io/notebook-image=true'
```

Now login with your OpenShift credentials. On the Spawner page, the Jupyter Spawn Image dropdown should contain an entry called _ml-workshop_. If it's not there, delete the _jupyterhub-XXXXXX_ pod and try again

- minio-ml-workshop: login with credentials _minio / minio123_
- ml-modeldb-webapp: no credentials needed
- odh-dashboard: not required for the workshop
- superset: login with credentials _admin / admin_

## Import assets

### Jenkins

Login to Jenkins as described previously and choose New Item as shown.
![](https://github.com/masoodfaisal/ml-workshop/blob/main/docs/images/33-jenkins-new-item.png)

Name it _deploy-model_, select _Pipeline_ as shown and click *OK*:
![](https://github.com/masoodfaisal/ml-workshop/blob/main/docs/images/34-new-item-deploy-model.png)

Now go ahead and click _This project is parameterized_ and add the 2 String parameters _namespace_ (with default ml-workshop) and *experiment_id* as shown:
![](https://github.com/masoodfaisal/ml-workshop/blob/main/docs/images/35-pipeline-param.png)

Don't save it yet.

Scroll down to the _Pipeline_ section. Inside the _Script_ box, you need to paste in a _Jenkinsfile_. You have two options


1 - if you are forking this repo for a classroom scenario, modify the 3 lines shown on blob/main/jenkins-pipeline/model/Jenkinsfile with **your** Minio route (that you tested above). Then commit that to your fork and pull from there.
![](https://github.com/masoodfaisal/ml-workshop/blob/main/docs/images/35-pipeline-change.png)


2 - Use [Jenkinsfile](https://raw.githubusercontent.com/masoodfaisal/ml-workshop/main/jenkins-pipeline/model/Jenkinsfile) as and make the 3 Minio changes in point 1 above - as shown and click *Save*:

![](https://github.com/masoodfaisal/ml-workshop/blob/main/docs/images/36-add-jenkinsfile.png)


Next, you need to ensure the Jenkins Service IPs are set. 
- Inside Jenkins, navigate to _Jenkins -> Manage Jenkins_
![](https://github.com/masoodfaisal/ml-workshop/blob/main/docs/images/36-jenkins-manage-1.png)

- Scroll down to _Manage Nodes and Clouds_
![](https://github.com/masoodfaisal/ml-workshop/blob/main/docs/images/36-jenkins-manage-2.png)

- Choose _Configure Clouds_
![](https://github.com/masoodfaisal/ml-workshop/blob/main/docs/images/36-jenkins-manage-3.png)

- Click _Kubernetes Cloud Details_
![](https://github.com/masoodfaisal/ml-workshop/blob/main/docs/images/36-jenkins-manage-4.png)

- Scroll down to _Jenkins URL_ and _Jenkins Tunnel_. They should look like this (Jenkins URL starting with _http_ and with port 8080, Jenkins Tunnel just an IP and port 50000)
![](https://github.com/masoodfaisal/ml-workshop/blob/main/docs/images/36-jenkins-manage-5.png)

- If not, open a new tab and go to Networking -> Services, filter on _Jenkins_ and get the 2 IPs and slot them into the previous section (_Jenkins URL_ and _Jenkins Tunnel_) remembering to begin _Jenkins URL_ with a _http_ and save there.
![](https://github.com/masoodfaisal/ml-workshop/blob/main/docs/images/36-jenkins-services.png)

### Add raw CSV files to Minio S3 object storage

Login to Minio as described previously and choose the _rawdata_ bucket. Then drag the 2 files *Customer-Churn_P1.csv* and *Customer-Churn_P2.csv* to Minio ( located on your file system in $REPO_HOME/data ) as shown:

![](https://github.com/masoodfaisal/ml-workshop/blob/main/docs/images/37-drag-raw-files-to-minio.png)


--------------------------------------------------------------------------------------------------------


# Adding users to the workshop
We provide a sample 30 user setup: _user1_.._user30_ each with the password _openshift_
These have beeen populated to the file _users.htpasswd_ in this directory.
First we create a secret with those users and their password:
```
cd $REPO_HOME/docs
oc create secret generic htpass-secret --from-file=htpasswd=users.htpasswd -n openshift-config
```
We've created a custom resource that sets up this htpasswd mechanism on OpenShift - which we apply as follows:
```
oc apply -f htpasswd.cr
```

If you need to give the users access to their own namespace(project), say _userX-project_. We also need to give Jenkins (used for CICD) access to each user's project.
That can be done as follows:
```
for i in {1..30}
do
    oc new-project user$i-project
    oc adm policy add-role-to-user admin user$i -n user$i-project
    oc adm policy add-role-to-user admin user$i -n ml-workshop
    oc create sa seldon-manager -n user${i}-project
    oc adm policy add-cluster-role-to-user cluster-admin system:serviceaccount:user${i}-project:seldon-manager -n user${i}-project
    oc adm policy add-role-to-user admin system:serviceaccount:ml-workshop:jenkins-ml-jenkins -n user$i-project
done
oc adm policy add-cluster-role-to-user admin user29
```
Note also I added admin access for one of my users, user29.

If you need to create users with different credentials consult [this blog](https://medium.com/kubelancer-private-limited/create-users-on-openshift-4-dc5cfdf85661) - on which these instructions are based.



--------------------------------------------------------------------------------------------------------


# Updating Tool URLs
As mentioned above, if you are running this as a workshop, it is recommended you fork this repo.  The reason is, after you install the tools, your OpenShift Service IP addresses for various tools will be different for each installation. It is recommended for simplicity, that you update yours with your cluster's values, so your students don't have to.
If you are forking the repo, you'll need to update the docs (all .md files in this directory) and replace all instances of https://github.com/masoodfaisal/ml-workshop with https://github.com/**YOUR_REPO**/ml-workshop

You need to find **your** IP addresses for  
a) the Minio object storage Service which we'll refer to as MINIO_IP, and 

b) the Verta.ai model repository Service which we'll refer to as VERTA_IP.

MINIO_IP and VERTA_IP are retrieved by navigating to Networking -> Services and locate the IP of their respective Services (verta being named _ml-modeldb-webapp_):
![](https://github.com/masoodfaisal/ml-workshop/blob/main/docs/images/38-service_ips.png)

MINIO_IP needs to be substituted in one file */notebook/Merge_Data.ipynb*. Open that file and search for _:9000_. Replace the IP that precedes the single instance of _:9000_ with your MINIO_IP.

VERTA_IP needs to be substituted in two files */notebook/Model_Experiments.ipynb* and */notebook/Train_Model.ipynb*. Open each of those files and search for _:3000_. Replace the IP that precedes the single instance of _:3000_ in each file with your VERTA_IP.

Save each of the three files and commit to your fork of this repository.

