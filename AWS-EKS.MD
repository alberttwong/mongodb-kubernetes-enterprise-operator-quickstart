# Quickstart of installing the MongoDB Enterprise Operator on AWS EKS

Here is a quickstart of installing the MongoDB Operator on AWS EKS.  There are steps needed for cluster wide and additional steps for each namespace/project.

# Architecture

mongodb namespace contains: operator

mongodb namespace contains: ops manager

ecommerce, portal, your-app, etc, etc namespace contains: replica set

# Instructions

## Setup AWS EKS

1. `eksctl create cluster --region=us-west-2`.

## Install the MongoDB Enterprise Operator

1. `kubectl create namespace mongodb`
1. `kubectl apply -f https://raw.githubusercontent.com/mongodb/mongodb-enterprise-kubernetes/master/crds.yaml`
1. `kubectl apply -f https://raw.githubusercontent.com/mongodb/mongodb-enterprise-kubernetes/master/mongodb-enterprise.yaml -n mongodb`
1. Verify `kubectl get pods -n mongodb`

## Install Ops Manager

1. `kubectl create secret generic om-admin-secret --from-literal=Username="jane.doe@example.com" --from-literal=Password="^DSghgds&*797JGHghg&^9ghGKG^*(7986" --from-literal=FirstName="Jane" --from-literal=LastName="Doe" -n mongodb`
1. Create OM
    ```
    ---
    apiVersion: mongodb.com/v1
    kind: MongoDBOpsManager
    metadata:
      name: om
    spec:
      replicas: 1
      version: 4.2.4
      adminCredentials: om-admin-secret
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
        # LoadBalancer|NodePort
        type: NodePort
        # optional, corresponds to nodePort port
        port: 30700

      applicationDatabase:
        members: 3
        version: 4.2.0
        persistent: true
    ```
1. Create a k8s route so that users can access OM.   The operator doesn't create a route automatically.  You'll have to create the route manually.
1. Verify `kubectl get om -n mongodb -o yaml -w`

## Install a replica set

1. `kubectl config set-context --current --namespace=default`
1. `kubectl create secret generic mongodb-opsmanager-creds --from-literal="user=NTFANWDB" --from-literal="publicApiKey=e2719091-8810-441d-9186-f0a747382922"`
1. `kubectl create configmap mongodb-cloudmanager-orgid --from-literal="baseUrl=https://cloud.mongodb.com" --from-literal="orgId=5f9310f51de4a4465a4f2318"`
1. Create db admin user so that you can log into the replica set after it's been created.
1. Tie the k8s user account to the mongodb database
1. Apply yaml
1. `kubectl get mdb repl -n default -o yaml -w`


## Connnect to replica set

1. Test if you can connect with the mongodb CLI. 
   1. If you want to test within the k8s namespace, you'll need something like mongodb toolbox (https://hub.docker.com/r/atwong/tool-box) to run `mongo --host my-replica-set4-0.my-replica-set4-svc.ecommerce.svc.cluster.local --port 27017 --username awong --password awong`
   1. `kubectl get svc -n default`, `kubectl port-forward svc/my-replica-set4-svc-external 27017:27017` and `mongo localhost:30091` 
1. Test with an Node.js application
1. Exposing MongoDB replica set outside of the k8s namespace via routes (Optional but not recommended)

## Delete MongoDB Operator

1. `kubectl delete namespace mongodb`
