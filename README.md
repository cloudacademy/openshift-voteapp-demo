# OpenShift VoteApp Deployment Instructions

The following instructions are used to demonstrate how to provision an OpenShift 4 cluster on AWS, and then to deploy a cloud native application into it.

![OpenShiftDeployment](/docs/OpenShiftDeployment.png)

# STEP1:

Review and familiarize yourself with the OpenShift documentation found here

https://www.openshift.com/

Log into the ```cloud.redhat.com``` portal - ensure that you have access to retrieve your OpenShift 4 cluster pull secret

https://cloud.redhat.com/openshift/install

# STEP2:

Download the ```openshift-install``` installer and setup

https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/

```
curl -O https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-install-mac-4.2.4.tar.gz
tar -xvf openshift-install-mac-4.2.4.tar.gz
cp openshift-install /usr/local/bin/
which openshift-install
```

```
openshift-install
openshift-install version
```

# STEP3:

Generate and scaffold a ```install-config.yaml``` file

```
openshift-install create install-config
```

# STEP4:

Over write the ```install-config.yaml``` with a demo cluster config
Note: this is for demo purposes, and is configured as follows:
1. SINGLE AZ zone configured to keep running costs down
2. 3 x m4.xlarge cluster master nodes
3. 1 x m4.xlarge compute node

Generate the SSH public key from an existing private key in your possession

```
ssh-keygen -y -f ~/.ssh/OpenShiftUSEast1.pem
```

Copy the entire key string into the ```sshKey``` field within the ```install-config.yaml`` file

Retrieve the pullSecret from the RedHat OpenShift cluster web portal:
 - https://cloud.redhat.com/openshift/install
 - copy this into the ```pullSecret``` field within the ```install-config.yaml``` file

```
cat > install-config.yaml << EOF
apiVersion: v1
baseDomain: democloudinc.com
controlPlane:
  hyperthreading: Enabled
  name: master
  platform:
    aws:
      zones:
      - us-east-1a
      #- us-east-1b
      #- us-east-1c
      rootVolume:
        size: 100
        type: gp2
      type: m4.xlarge
  replicas: 3
compute:
- hyperthreading: Enabled
  name: worker
  platform:
    aws:
      zones:
      - us-east-1a
      #- us-east-1b
      #- us-east-1c
      rootVolume:
        size: 100
        type: gp2
      type: m4.xlarge
  replicas: 1
metadata:
  name: openshift
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineCIDR: 192.168.0.0/20
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
platform:
  aws:
    region: us-east-1
    userTags:
      adminContact: Jeremy
      costCenter: 123
      email: jeremy.cook@cloudacademy.com
pullSecret: ''
sshKey: 
EOF
```

Create the cluster

```
openshift-install create cluster --log-level debug
```

Notes:
1. This takes between 10-20 minutes to complete so go make a coffee!!
2. Use the "--log-level debug" to see the provisioning activity - useful to detect errors earlier

# STEP5:

Open the web console and login
* https://console-openshift-console.apps.openshift.democloudinc.com
* https://oauth-openshift.apps.openshift.democloudinc.com/oauth/token/display

Notes:
1. URL maybe different depending on the values used and configured within the ```install-config.yaml```
2. The auth folder contains credentials for logging in

# STEP6:

Install the ```oc``` command:

```
curl -O https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-mac-4.2.4.tar.gz
tar -xvf openshift-client-mac-4.2.4.tar.gz
cp oc /usr/local/bin/
which oc
```

```
oc
oc version
```

Alternatively for MacOS use

```
brew install openshift-cli
```

# STEP7:

Test the client cluster authencation

```
oc whoami
oc whoami -t
oc get user
```

# STEP8:

Install and setup an IDP htpasswd file and import 

Create a new ```users.htpasswd``` file

```
htpasswd -c -B -b users.htpasswd jeremy blah
```

Import ```users.htpasswd``` via the web console

Update the permissions on the new user - give cluster admin permissions

```
oc adm policy add-cluster-role-to-user cluster-admin jeremy
oc login -u jeremy -p blah
oc whoami
oc get user
oc get scc
```

Delete the temp ```kubeadmin``` user

```
oc delete secrets kubeadmin -n kube-system
```

After you define an identity provider and create a new cluster-admin user, you can remove the kubeadmin to improve cluster security.
You must have configured at least one identity provider.
You must have added the cluster-admin role to a user.
You must be logged in as an administrator.

# STEP9:

Create the ```cloudacademy``` project (namespace)

```
oc new-project cloudacademy
oc status
```

or

```
cat <<EOF | oc create -f -
kind: Project
apiVersion: project.openshift.io/v1
metadata:
  name: cloudacademy
  annotations:
    openshift.io/description: ''
    openshift.io/display-name: CloudAcademy
EOF
```

# STEP10:

Create a new StorageClass named ```ebs```

```
cat <<EOF | oc create -f -
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: ebs
provisioner: kubernetes.io/aws-ebs
parameters:
  fsType: ext4
  type: gp2
reclaimPolicy: Retain
volumeBindingMode: Immediate
EOF
```

# STEP11:

Create a new Mongo StatefulSet name ```mongo```

```
cat <<EOF | oc create -f -
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: mongo
  namespace: cloudacademy
spec:
  serviceName: mongo
  replicas: 3
  template:
    metadata:
      labels:
        role: db
        env: demo
        replicaset: rs0.main
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: replicaset
                  operator: In
                  values:
                  - rs0.main
              topologyKey: kubernetes.io/hostname
      terminationGracePeriodSeconds: 10
      containers:
        - name: mongo
          image: mongo
          command:
            - "numactl"
            - "--interleave=all"
            - "mongod"
            - "--wiredTigerCacheSizeGB"
            - "0.1"
            - "--bind_ip"
            - "0.0.0.0"
            - "--replSet"
            - "rs0"
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: mongodb-persistent-storage-claim
              mountPath: /data/db
  volumeClaimTemplates:
    - metadata:
        name: mongodb-persistent-storage-claim
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: ebs
        resources:
          requests:
            storage: 0.5Gi
EOF
```

Examine the Mongo Pods, PersistentVols, and PersistentVolumeClaims

```
oc get pod,pv,pvc
```

# STEP12:

Create a new Headless Service for Mongo named ```mongo```

```
cat <<EOF | oc create -f -
apiVersion: v1
kind: Service
metadata:
  name: mongo
  namespace: cloudacademy
  labels:
    role: db
    env: demo
spec:
  ports:
  - port: 27017
    targetPort: 27017
  clusterIP: None
  selector:
    role: db
EOF
```

Examine the Mongo Headless Service

```
oc get svc
```

Examine the DNS records for the Mongo Headless Service

```
oc run --generator=run-pod/v1 --rm utils -it --image eddiehale/utils bash
```

Within the new utils container run the following DNS queries

```
host mongo
for i in {0..2}; do host mongo-$i.mongo; done
```

# STEP13:

Initialise the Mongo database replica set

```
cat > db.init.js << EOF
rs.initiate();
rs.add("mongo-1.mongo:27017");
rs.add("mongo-2.mongo:27017");
cfg = rs.conf();
cfg.members[0].host = "mongo-0.mongo:27017";
rs.reconfig(cfg, {force: true});
sleep(5000);
EOF
```

```
oc rsh mongo-0 mongo < db.init.js
oc rsh mongo-0 mongo --eval "rs.status()"
```

# STEP14:

Load the initial voting app data into the Mongo database

```
cat > db.load.js << EOF
use langdb;
db.languages.insert({"name" : "go", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 16, "compiled" : true, "homepage" : "https://golang.org", "download" : "https://golang.org/dl/", "votes" : 0}});
db.languages.insert({"name" : "java", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 2, "compiled" : true, "homepage" : "https://www.java.com/en/", "download" : "https://www.java.com/en/download/", "votes" : 0}});
db.languages.insert({"name" : "nodejs", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 30, "compiled" : false, "homepage" : "https://nodejs.org/en/", "download" : "https://nodejs.org/en/download/", "votes" : 0}});
db.languages.find().pretty();
EOF
```

```
oc rsh mongo-0 mongo < db.load.js
oc rsh mongo-0 mongo langdb --eval "db.languages.find().pretty()"
```

# STEP15:

Deploy the API using:

```
cat <<EOF | oc apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: cloudacademy
  labels:
    role: api
    env: demo
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 25%
  selector:
    matchLabels:
      role: api
  template:
    metadata:
      labels:
        role: api
    spec:
      containers:
      - name: api
        image: cloudacademydevops/api:v1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
        livenessProbe:
          httpGet:
            path: /ok
            port: 8080
          initialDelaySeconds: 2
          periodSeconds: 5
        readinessProbe:
          httpGet:
             path: /ok
             port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          successThreshold: 1
---
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: cloudacademy
  labels:
    role: api
    env: demo
spec:
  ports:
   - protocol: TCP
     port: 8080
  selector:
    role: api
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: api
  namespace: cloudacademy
spec:
  path: /
  to:
    kind: Service
    name: api
  port:
    targetPort: 8080
EOF
```

# STEP16:

Examine the status of the API deployment
Examine the pods to confirm that they are up and running
Examine api pod log to see mongo db connected message

```
oc rollout status deployment api
oc get pods
oc logs API_POD_NAME_HERE
oc get svc
oc get route
```

Test the API route url - test the ```/ok```, ```/languages```, and ```/languages/{name}``` endpoints

```
curl -s http://API_ROUTE_URL/ok
curl -s http://API_ROUTE_URL/languages | jq .
curl -s http://API_ROUTE_URL/languages/go | jq .
curl -s http://API_ROUTE_URL/languages/java | jq .
curl -s http://API_ROUTE_URL/languages/nodejs | jq .
```

# STEP17:

Install and setup the S2I executable

https://github.com/openshift/source-to-image/releases/

```
curl -O --location https://github.com/openshift/source-to-image/releases/download/v1.2.0/source-to-image-v1.2.0-2a579ecd-darwin-amd64.tar.gz
tar -xvf source-to-image-v1.2.0-2a579ecd-darwin-amd64.tar.gz
cp s2i /usr/local/bin/
which s2i
```

Alternatively for MacOS use

```
brew install source-to-image
```

=============================

# STEP18:

Try out the S2I generator

```
s2i create xyzbuilder s2i-xyzbuilder
```

Examine the new ```s2i-xyzbuilder``` directory structure

```
cd s2i-xyzbuilder
tree s2i-xyzbuilder
```

Examine the ```Makefile```, ```Dockerfile```, ```assemble```, and ```run``` files

```
cat Makefile
cat Dockerfile
cat s2i/bin/assemble
cat s2i/bin/run
```

Create the builder image

make

Examine the newly created docker image

```
docker images | grep xyzbuilder
```

# STEP19:

Git clone the example frontend builder S2I repo

```
git clone https://github.com/cloudacademy/openshift-s2i-frontendbuilder.git
```

Examine the new ```openshift-s2i-frontendbuilder``` directory structure

```
cd openshift-s2i-frontendbuilder/
tree
```

Examine the ```Makefile```, ```Dockerfile```, ```assemble```, and ```run``` files

```
cat Dockerfile
cat s2i/bin/assemble
cat s2i/bin/run
cat Makefile
```

Create the builder image

```
make
```

Examine the newly created docker image with tag ```cloudacademydevops/frontendbuilder```

```
docker images | grep cloudacademydevops/frontendbuilder
```

# STEP20:

Create the runtime frontend image, referencing the GitHub hosted ```openshift-voteapp-frontend-react``` repo

```
s2i build https://github.com/cloudacademy/openshift-voteapp-frontend-react cloudacademydevops/frontendbuilder cloudacademydevops/frontend:demo-v1
```

Examine the docker images for the ```cloudacademydevops/frontend:demo-v1``` image

```
docker images | grep cloudacademydevops/frontend:demo-v1
```

Test locally the runtime ```cloudacademydevops/frontend:demo-v1``` docker image

```
docker run -e REACT_APP_APIHOSTPORT=xxxxxxxxxxxx -it --rm --name test -p 8080:8080 cloudacademydevops/frontend:demo-v1
```

Browse to the application and ensure that the ```REACT_APP_APIHOSTPORT``` environment variable has been correctly injected and is being used for the AJAX calls to the API

# STEP21:

Upload the S2I Frontend builder image into the ```cloudacademydevops``` DockerHub repo

Note:
1. Use your own DockerHub repo

```
docker login
docker push cloudacademydevops/frontendbuilder:latest
```

Now import it from DockerHub into the OpenShift cluster

```
oc import-image docker.io/cloudacademydevops/frontendbuilder:latest --confirm -n cloudacademy
```

# STEP22:

Create a new BuildConfig for the frontend

Note:
1. Consider forking the GitHub repo: https://github.com/cloudacademy/openshift-voteapp-frontend-react
2. Then update the gti url field with your forked GitHub URL
3. Update the secret value - use your own secret

```
cat <<EOF | oc apply -f -
kind: BuildConfig
apiVersion: build.openshift.io/v1
metadata:
  name: frontend
  namespace: cloudacademy
spec:
  output:
    to:
      kind: ImageStreamTag
      name: 'frontend:latest'
  resources: {}
  successfulBuildsHistoryLimit: 5
  failedBuildsHistoryLimit: 5
  strategy:
    type: Source
    sourceStrategy:
      from:
        kind: ImageStreamTag
        namespace: cloudacademy
        name: 'frontendbuilder:latest'
  postCommit: {}
  source:
    type: Git
    git:
      uri: 'https://github.com/cloudacademy/openshift-voteapp-frontend-react'
      ref: master
  triggers:
    - type: GitHub
      github:
        secret: supersecretgoeshere
  runPolicy: Serial
EOF
```

Examine current build configs

```
oc get bc
```

# STEP23:

Create a new ImageStream named frontend - this will be populated with frontend images created by the frontend buildconfig

```
oc create imagestream frontend
```

Examine all current image storageClassName

```
oc get is
```

# STEP24:

Trigger a manual build for the frontend and tail the resulting build log

```
oc start-build frontend --follow
```

or 

```
oc get builds
oc logs -f build/BUILD_NAME
```

# STEP25:

Create a new frontend DeploymentConfig
Notes: 
1. Before deploying make sure the ```REACT_APP_APIHOSTPORT``` environment var is updated with the correct value, use ```oc get route api``` to discover it

```
cat <<EOF | oc apply -f -
kind: DeploymentConfig
apiVersion: apps.openshift.io/v1
metadata:
  name: frontend
  namespace: cloudacademy
  labels:
    env: demo
    role: frontend
spec:
  strategy:
    type: Rolling
    rollingParams:
      updatePeriodSeconds: 1
      intervalSeconds: 1
      timeoutSeconds: 600
      maxUnavailable: 25%
      maxSurge: 25%
  triggers:
    - type: ConfigChange
    - type: ImageChange
      imageChangeParams:
        automatic: true
        containerNames:
          - frontend
        from:
          kind: ImageStreamTag
          namespace: cloudacademy
          name: frontend:latest
  replicas: 4
  selector:
    role: frontend
  template:
    metadata:
      labels:
        role: frontend
    spec:
      containers:
        - resources: {}
          readinessProbe:
            httpGet:
              path: /
              port: 8080
              scheme: HTTP
            initialDelaySeconds: 5
            timeoutSeconds: 1
            periodSeconds: 5
            successThreshold: 1
            failureThreshold: 3
          terminationMessagePath: /dev/termination-log
          name: frontend
          livenessProbe:
            httpGet:
              path: /
              port: 8080
              scheme: HTTP
            initialDelaySeconds: 2
            timeoutSeconds: 1
            periodSeconds: 5
            successThreshold: 1
            failureThreshold: 3
          env:
            - name: REACT_APP_APIHOSTPORT
              value: api-cloudacademy.apps.openshift.democloudinc.com
          ports:
            - containerPort: 8080
              protocol: TCP
          imagePullPolicy: IfNotPresent
          terminationMessagePolicy: File
      restartPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: cloudacademy
  labels:
    role: frontend
    env: demo
spec:
  ports:
   - protocol: TCP
     port: 8080
  selector:
    role: frontend
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: frontend
  namespace: cloudacademy
spec:
  path: /
  to:
    kind: Service
    name: frontend
  port:
    targetPort: 8080
EOF
```

Examine the rollout of the DeploymentConfig

```
oc rollout status deploymentconfig frontend
oc get pods
```

Use the chrome browser and test the application via the frontend route url

```
oc route get frontend
```

# STEP26:

Tail the API pod logs

```
oc get pods
oc logs API_POD_NAME --tail 50 --follow
```

Check the updated vote count held within the Mongo database

```
oc get pods
oc rsh mongo-0 mongo langdb --eval "db.languages.find().pretty()"
```

# STEP27:

Configure auto build and deployments for the Frontend pods

```
oc get bc
oc describe bc frontend
```

Copy the Github webhook url, this can also be found and copied from within the Openshift web admin console

Will look something like the following:

```
https://api.openshift.democloudinc.com:6443/apis/build.openshift.io/v1/namespaces/blah/buildconfigs/frontend/webhooks/supersecretgoeshere/github
```

Login into the GitHub repo: https://github.com/cloudacademy/openshift-voteapp-frontend-react

GitHub Webhook Setup Instructions:
1. Under Settings->Webhooks, click the "Add Webhook" button
2. Enter the buildconfig URL into the Payload URL
3. Set Content-Type to be "application/json"
4. Leave the Secret field empty
5. Disable SSL verification - during testing and demos
6. Then click the "Add webhook" button at the bottom 

Confirm that the Webhook configuration has been successfully set

# STEP28:

Now perform an edit within the VoteApp codebase.

For example update the VoteApp.js file - update the version number in the ```CloudAcademy ❤ DevOps 2019 v2.10.1``` string:

```
openshift-voteapp-frontend-react/src/components/VoteApp.js

import React, { Component } from 'react';
import ProgrammingLanguage from './ProgrammingLanguage';

class VoteApp extends Component {
  render () {
    return (
      <main role="main">
        <div class="jumbotron">
          <div class="container">
            <h1 class="display-3">Language Vote App v2</h1>
            &copy; CloudAcademy ❤ DevOps 2019 v2.10.1
          </div>
        </div>

        <div class="container">
          <div class="row">
            <div class="col-md-4">
              <ProgrammingLanguage id="go" logo="go.png"/>
            </div>
            <div class="col-md-4">
              <ProgrammingLanguage id="java" logo="java.png"/>
            </div>
            <div class="col-md-4">
              <ProgrammingLanguage id="nodejs" logo="nodejs.png"/>
            </div>
          </div>
        </div>
      </main>
    )
  }
}

export default VoteApp;
```

Save, commit, and push back into the repo

```
git add VoteApp.js
git commit -m "updated to trigger a build"
git push
```

# STEP29:

Check that a new OpenShift build has been triggered

```
oc get builds
```

Tail the new build

```
oc logs -f build/BUILD_NAME
```

Watch for the new automatic DeploymentConfig update triggered by the build

```
oc rollout status deploymentconfig frontend
oc get pods
```

Confirm the complete end-to-end build and deploy process has worked by 
refreshing the application within the browser!!

:+1: Result!!