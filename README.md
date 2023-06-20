# Backup a Percona MongoDB cluster with Kasten

This demo project show how to backup a MongoDB cluster managed by the persona operator with Kasten. 

## Motivation

Kasten backup your workloads with the storage by a single backup action on your namespace. 

We see more and more stateful workloads managed by operators because they become more complex
and all operations need to be automated. For instance with mongo you have sharding and replication
for high availability and horizontal scalability. 

The [Percona operator for MongoDB](https://docs.percona.com/percona-operator-for-mongodb/index.html)
manage this conplexity by letting you define your mongodb desired state through a custom resource : 

```
apiVersion: psmdb.percona.com/v1
kind: PerconaServerMongoDB
metadata:
  name: my-cluster-name
  finalizers:
    - delete-psmdb-pods-in-order
spec:
  crVersion: 1.13.0
  image: percona/percona-server-mongodb:5.0.11-10
  updateStrategy: SmartUpdate  
  replsets:
  - name: rs0
    size: 3 # <---------------------------------------------- 3 replicas servers 
    resources:
      limits:
        cpu: "300m"
        memory: "0.5G"
      requests:
        cpu: "300m"
        memory: "0.5G"
    volumeSpec:
      persistentVolumeClaim:
        resources:
          requests:
            storage: 3Gi
  sharding:
    enabled: true
    configsvrReplSet:
      size: 3   # <------------------------------------------ 3 conf servers 
...
```

This custom resource generate derived resources wich are the actual workloads : 

![Custom Resource and derived resources](./images/CR-and-Derived-Resources.png)

In this situation we discover an operator dichotomy : 
1. The control plane : The operator controller and the Custom Resource Definition 
2. The custom resources : Define your stateful workload
3. The derived resources : All the resources created by the operator controller 

![Operator Dicothomy](./images/operator-dichotomy.png)

## Then what should we backup and how ?

### The control plane ? 

You can backup  the control plane for forensic reason but you won't restore it most of the time. 

This is because the control plane is often doing low-level operation, has to get high level of 
authorization, depends of the kubertes version of other dependencies and for this reason you 
prefer to reinstall rather than restore the control plane.

### The custom resource ?

Indeed the custom resource is what you need to backup, and when you back it up you have the 
opportunity to invoke and manage the operator backup routine. 

Here Kasten become very powerful, because it can be configured to automatically trigger the 
operator backup routine for this specific kind of Custom resource through the call of a specific 
blueprint. In the same namespace there could be different custom resource and different blueprint
accordingly will be called. 

### The derived resource ? 

The derived resource could be backed up and restored. It really depends of the operator capacity to 
reconcile between the custom resource and the derived resource when restored. 

Usually an operator work on a custom resource and create the derived resource from it, if it maintains
information in memory then in a restore situation the operator has to handle derived resource that he 
do not "remember" having created. 

Hence handling restored derived resource really depends of the operator implementation. 

## Approach for The Percona Server Mongodb Operator 

For the Percona Mongodb Operator both approach were tested: 
- Backing and Restoring only the custom resource with the automatic invokation of a blueprint 
- Backing and Restoring the custom resource and the derived resources

They both worked, we get successfull backup and restore but the second approach won't be supported
by Percona, and on large dataset you may be unaware of unconsistencies, having the database restarting 
does not mean it's guaranteed with zero data loss. On the other hand the first approach is using the 
official backup and restore api supported by Percona.


# Let's test it 

## Create the cluster with the bucket configuration for the backup 

In this demo the backup are sent to an AWS S3 bucket but the operator support Azure blob and other S3 compatible service.

Let's define for the S3 bucket the backup location, you'll need to change it line 465-472 in `cr.yaml`.
- `AWS_S3_ACCESS_KEY_ID` and `AWS_S3_SECRET_ACCESS_KEY` according to your setting.
- `REGION` the region of your bucket for instance eu-west-3
- `BUCKET_NAME` the name of you bucket for instance my-bucket
- `PREFIX` the prefix of the backup in the bucket for instance mongodb/my-cluster-name

Let's create in this order : 
- the namespace, 
- the operator,
- the secret for the S3 backup location
- the cluster 

```
kubectl create ns mcourcy-mongodb-percona
kubectl project mcourcy-mongodb-percona
kubectl apply --server-side -f https://raw.githubusercontent.com/percona/percona-server-mongodb-operator/v1.14.0/deploy/bundle.yaml
kubectl create secret generic s3-secret --from-literal AWS_ACCESS_KEY_ID=$AWS_S3_ACCESS_KEY_ID --from-literal AWS_SECRET_ACCESS_KEY=$AWS_S3_SECRET_ACCESS_KEY
kubectl create -f cr.yaml
```

Wait for all the pods to be ready
```
kubectl get pods
```

check the status of the mongodb cluster
```
kubectl get psmdb
```

## Insert and read data in the cluster 

Once MongoDB is running, you can populate it with some data. Let's add a collection called "ticker":

```bash
MONGODB_DATABASE_ADMIN_PASSWORD=$(kubectl get secret my-cluster-name-secrets -ojsonpath='{.data.MONGODB_DATABASE_ADMIN_PASSWORD}'|base64 -d)
MONGODB_DATABASE_ADMIN_USER=$(kubectl get secret my-cluster-name-secrets -ojsonpath='{.data.MONGODB_DATABASE_ADMIN_USER}'|base64 -d)
kubectl run -i --rm --tty percona-client \
    --image=percona/percona-server-mongodb:6.0.4-3 \
    --env=MONGODB_DATABASE_ADMIN_PASSWORD=$MONGODB_DATABASE_ADMIN_PASSWORD \
    --env=MONGODB_DATABASE_ADMIN_USER=$MONGODB_DATABASE_ADMIN_USER \
    --restart=Never \
    -- bash -il
mongosh "mongodb://$MONGODB_DATABASE_ADMIN_USER:$MONGODB_DATABASE_ADMIN_PASSWORD@my-cluster-name-mongos/admin?ssl=false"
```

Insert some data 
```
db.createCollection("ticker")
db.ticker.insert(  {createdAt: new Date(), randomdata: "qstygshgqsfxxtqsfgqfhjqhsj"} )
db.ticker.insert(  {createdAt: new Date(), randomdata: "qstygshgqsfxxtqsfgqfhjqhsj"} )
db.ticker.insert(  {createdAt: new Date(), randomdata: "qstygshgqsfxxtqsfgqfhjqhsj"} )
db.ticker.insert(  {createdAt: new Date(), randomdata: "qstygshgqsfxxtqsfgqfhjqhsj"} )
db.ticker.find({}).sort({createdAt:-1}).limit(1)
```

Create a ticker pod that add new entry every seconds so that you compare the date of the backup 
and the date of the last entries.

```
MONGODB_DATABASE_ADMIN_PASSWORD=$(kubectl get secret my-cluster-name-secrets -ojsonpath='{.data.MONGODB_DATABASE_ADMIN_PASSWORD}'|base64 -d)
MONGODB_DATABASE_ADMIN_USER=$(kubectl get secret my-cluster-name-secrets -ojsonpath='{.data.MONGODB_DATABASE_ADMIN_USER}'|base64 -d)
kubectl run percona-ticker \
    --image=percona/percona-server-mongodb:6.0.4-3 \
    --env=MONGODB_DATABASE_ADMIN_PASSWORD=$MONGODB_DATABASE_ADMIN_PASSWORD \
    --env=MONGODB_DATABASE_ADMIN_USER=$MONGODB_DATABASE_ADMIN_USER \
    -- bash -c "while true; do mongosh \"mongodb://$MONGODB_DATABASE_ADMIN_USER:$MONGODB_DATABASE_ADMIN_PASSWORD@my-cluster-name-mongos/admin?ssl=false\" --eval 'db.ticker.insert(  {createdAt: new Date(), randomdata: \"qstygshgqsfxxtqsfgqfhjqhsk\"} )'; sleep 1; done"
```

Follow the insertion of the data 
```
kubectl logs percona-ticker -f 
```

Reuse the percona-client to check you have a new entry every second, execute this command multiple times.
```
db.ticker.find({}).sort({createdAt:-1}).limit(1)
```


## Install and configure the blueprint 

```
kubectl create -f psmdb-bp.yaml 
kubectl annotate psmdb my-cluster-name kanister.kasten.io/blueprint=psmdb-bp 
```

## Create a backup policy 

Create a backup policy on the namespace mcourcy-mongodb-percona 
- define both an export and kanister profile 
- Frequency on-demand 

And run it once.

At the end of the policy run you should see a new perconaservermongodbbackup object 
```
kubectl get psmdb-backup
```

# Simulate disaster and disaster recovery 

## Case 1. You still have the namespace but you want to recover only the database

To simulate a disaster let's delete the cluster and all its pvc 

```
kubectl delete psmdb --all
kubectl delete po percona-ticker
kubectl delete pvc --all 
```

Now go to the restore point and restore by just restoring : 
- the psmdb


At the end of the process you should see a psmdb-restore
```
kubectl get psmdb-restore
```

## Case 2. You lose the namespace or recover on another cluster

Import the restore point with an importPolicy 

recreate the namespace and reinstall the operator
```
kubectl create ns mcourcy-mongodb-percona
kubectl config set-context --current --namespace=mcourcy-mongodb-percona
kubectl apply --server-side -f https://raw.githubusercontent.com/percona/percona-server-mongodb-operator/v1.14.0/deploy/bundle.yaml
```

Reinstall the blueprint but without the restore action 
If it's the same cluster 
```
kubectl replace -f psmdb-bp-without-restore.yaml
```
If it's a new cluster 
```
kubectl create -f psmdb-bp-without-restore.yaml
```


Now go to the restore point and restore by just restoring : 
- the psmdb, 
- the secret s3-secret

Restore by recreating a restore object directly you need to obtain the destination from 
your previous backup or by listing the content of you bucket, here is an example 
```
apiVersion: psmdb.percona.com/v1
kind: PerconaServerMongoDBRestore
metadata:
  name: restore-manual
spec:
  clusterName: my-cluster-name
  backupSource:
    destination: s3://mcourcy-feesh/demo/mcourcy/percona/mongodb/my-cluster-name/2023-04-03T07:46:17Z 
    s3:
      credentialsSecret: s3-secret
      region: eu-west-3
      bucket: mcourcy-feesh
      prefix: ""
```

# Uninstall the demo

Remove completly the demo  by executing 

```
kubectl delete psmdb-restore --all
kubectl delete psmdb-backup --all
kubectl delete psmdb --all
kubectl delete -f https://raw.githubusercontent.com/percona/percona-server-mongodb-operator/v1.14.0/deploy/bundle.yaml
kubectl delete ns mcourcy-mongodb-percona
kubectl delete -f psmdb-bp.yaml
```