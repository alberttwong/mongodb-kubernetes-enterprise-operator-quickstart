# mongodb-kubernetes-enterprise-operator-quickstart

Here is a quickstart of installing the MongoDB Operator for OpenShift.  There are steps needed for cluster wide and additional steps for each namespace/project.

# Architecture

openshift-operator namespace contains: operator

mongodb namespace contains: ops manager

ecommerce, portal, your-app, etc, etc namespace contains: replica set

# Install the MongoDB Enterprise Operator via OperatorHub (one time install)

1. Install the MongoDB Enterprise Operator via OperatorHub.   Install with defaults (watch all namespace, automatic update)
2. Check the operator pod logs to see if it's installed correctly.

# Install Ops Manager
Prereq: Operator is installed.

1. Create a new project.  `oc new-project mongodb`
2. Execute the 1 command to create the username/passwd to log into Ops Manager running in the kube cluster. You do this for each mongodb cluster. I use the OCP CLI.

`
oc create secret generic ops-manager-admin-user-credentials --from-literal=Username="albert.wong@mongodb.com" --from-literal=Password="Admin1234$" --from-literal=FirstName="Albert" --from-literal=LastName="Wong"
`

```
---
apiVersion: mongodb.com/v1
kind: MongoDBOpsManager
metadata:
  name: om
spec:
  replicas: 1
  version: 4.2.4
  adminCredentials: ops-manager-admin-user-credentials
  backup:
    enabled: false
  configuration:
    mms.ignoreInitialUiSetup: "true"
    mms.fromEmailAddr: "jane.doe@example.com"
    mms.replyToEmailAddr: "jane.doe@example.com"
    mms.adminEmailAddr: "jane.doe@example.com"
    mms.emailDaoClass: "com.xgen.svc.core.dao.email.JavaEmailDao"
    mms.mail.transport: "smtp"
    mms.mail.hostname: "example.com"
    mms.mail.port: "25"
    mms.security.allowCORS: "false"
  externalConnectivity:
    # |NodePort
    type: LoadBalancer
  applicationDatabase:
    members: 3
    version: 4.2.0
    persistent: true
``` 

# Install a Replica set
The assumption is that you have a project (ecommerce_ that has already been created and you want to "add" a MongoDB replica set into this project.

1. Change to the correct project.  `oc project ecommerce`
2. Create all mongoDB k8s service accounts 

```
---
# Source: mongodb-enterprise-operator/templates/database-roles.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mongodb-enterprise-appdb
---
# Source: mongodb-enterprise-operator/templates/database-roles.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mongodb-enterprise-database-pods
---
# Source: mongodb-enterprise-operator/templates/database-roles.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mongodb-enterprise-ops-manager
---
# Source: mongodb-enterprise-operator/templates/database-roles.yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: mongodb-enterprise-appdb
rules:
  - apiGroups:
      - ""
    resources:
      - secrets
    verbs:
      - get
---
# Source: mongodb-enterprise-operator/templates/database-roles.yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: mongodb-enterprise-appdb
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: mongodb-enterprise-appdb
subjects:
  - kind: ServiceAccount
    name: mongodb-enterprise-appdb
```

3. Execute the 2 commands to create connectivity to cloud.mongodb.com Cloud Manager.  It is assumed that you have already created in Organization in MongoDB Cloud Manager and created an API key for the operator to use.   If you also know the IP address of your OCP cluster, you can whitelist it now.  If not, you can come back to this step and update the IP whitelist.

```
oc create secret generic mongodb-opsmanager-creds --from-literal="user=NTFANWDB" --from-literal="publicApiKey=e2719091-8810-441d-9186-f0a747382922"
oc create configmap mongodb-cloudmanager-orgid --from-literal="baseUrl=https://cloud.mongodb.com" --from-literal="orgId=5f9310f51de4a4465a4f2318"
```

4. Create db admin user so that you can log into the replica set after it's been created.

```
---
apiVersion: v1
kind: Secret
metadata:
  name: awong-password
  # corresponds to user.spec.passwordSecretKeyRef.name
type: Opaque
stringData:
  password: awong
  # corresponds to user.spec.passwordSecretKeyRef.key
...
```

5. Tie the k8s user account to the mongodb database

```
---
apiVersion: mongodb.com/v1
kind: MongoDBUser
metadata:
  name: mms-scram-awong
spec:
  passwordSecretKeyRef:
    name: awong-password
    # Match to metadata.name of the User Secret
    key: password
  username: "awong"
  db: "admin" #
  mongodbResourceRef:
    name: "my-replica-set4"
    # Match to MongoDB resource using authenticaiton
  roles:
    - db: admin
      name: clusterAdmin
    - db: admin
      name: dbAdminAnyDatabase
    - db: admin
      name: userAdminAnyDatabase
    - db: admin
      name: readWriteAnyDatabase
...
```

6. yaml for provisioning should look like this. This ties the mongodb-opsmanager-creds and mongodb-cloudmanager-orgid to the "my-replica-set4" mongodb cluster within your kube cluster in project/namespace "ecommerce". I use the "import YAML" UI in OCP console.

```
apiVersion: mongodb.com/v1
kind: MongoDB
metadata:
  name: my-replica-set4
spec:
  security:
    authentication:
      enabled : true
      modes: ["SCRAM"]
  credentials: mongodb-opsmanager-creds
  members: 3
  opsManager:
    configMapRef:
      name: mongodb-cloudmanager-orgid
  type: ReplicaSet
  version: 4.4.0-ent
  persistent: true
```

```
# TLS enabled in OpenShift doesn't currently work as of Jan 2021
apiVersion: mongodb.com/v1
kind: MongoDB
metadata:
  name: my-replica-set4
spec:
  security:
    tls:
      enabled: true
    authentication:
      enabled : true
      modes: ["SCRAM"]
  credentials: mongodb-opsmanager-creds
  members: 3
  opsManager:
    configMapRef:
      name: mongodb-cloudmanager-orgid
  type: ReplicaSet
  version: 4.4.0-ent
  persistent: true
```

If you run into errors, run `oc get mdb my-replica-set4 -o yaml -w` and at the bottom, it'll give you a status message. If it says that it cannot connect to Cloud Manager, it means that you need to update the API key whitelist.   If it says "not ready" or "reconciling", just give the operator more time to complete it's operations. 

7. Test if you can connect with the mongodb CLI.  
  
If you want to test within the k8s cluster, you'll need something like mongodb toolbox (https://hub.docker.com/r/atwong/tool-box) to run `mongo --host my-replica-set4-0.my-replica-set4-svc.ecommerce.svc.cluster.local --port 27017 --username awong --password awong`

If you want to test from your workstation, run `oc get svc`, find the correct k8s service `oc port-forward svc/my-replica-set4-svc-external 27017:27017` and then `mongo --host localhost --port 27017 --username awong --password awong`

8. Test with an Node.js application

Create new project. Add new application via git using https://github.com/alberttwong/nodejs-ex. Add the following k8s deployment variables

```
# MongoDB connection string
MONGO_URL = mongodb+srv://awong:awong@my-replica-set4-svc.ecommerce.svc.cluster.local/?ssl=false&replicaSet=my-replica-set4&authSource=admin
# This is a OCP Node.js base image option.  Allows me to hot deploy changes while shelled in the pod.
DEV_MODE = true
```

9. Exposing MongoDB via routes (Optional)

The operator doesn't create any OCP routes. This requires the use of k8s nodeport since MongoDB uses client side load balancing. For each replica set, you'll need to manually create 3 nodeports on the load balancer.  Once that is created, you'll create a connection string that contain those 3 nodeports. 
