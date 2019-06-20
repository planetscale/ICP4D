# ICP4D: PlanetScale Operator for Vitess

### Prerequisites

1. Obtain access to the PlanetScale Docker Repository.

Please sign up for an account on our [registry](https://registry.planetscale.com/) and send us the account name. We will process your request and authorize it, with a further email response back.

2. Install Vitess locally

This will allow you to use the Vitess command line tools, especially vtctlclient. On Linux, you can install Vitess locally by cloning the [planetscale/vitess-releases](https://github.com/planetscale/vitess-releases) GitHub repository, and running `install_latest.sh`. Alternatively, instructions for manually building Vitess from source can be found  [here](https://vitess.io/docs/tutorials/local/):

## Installation

### 1. Create a secret using your new PlanetScale registry credentials

`kubectl create secret docker-registry psregistry --docker-server=registry.planetscale.com --docker-username='your_username'  -—docker-email='your_email' --docker-password='your_password'`

Expected output:
`secret/psregistry created`

### 2. Deploy the operators

`kubectl create -f planetscale_operators.yaml`

Expected output:
```
role.rbac.authorization.k8s.io/planetscale-operator created
rolebinding.rbac.authorization.k8s.io/default-account-planetscale-operator created
clusterrole.rbac.authorization.k8s.io/planetscale-persistent-volume created
clusterrolebinding.rbac.authorization.k8s.io/default-account-planetscale-pv created
clusterrole.rbac.authorization.k8s.io/prometheus-operator created
clusterrolebinding.rbac.authorization.k8s.io/prometheus-operator created
clusterrole.rbac.authorization.k8s.io/prometheus created
clusterrolebinding.rbac.authorization.k8s.io/prometheus created
customresourcedefinition.apiextensions.k8s.io/psclusters.planetscale.com created
deployment.extensions/etcd-operator created
deployment.extensions/etcd-backup-operator created
deployment.extensions/etcd-restore-operator created
deployment.apps/planetscale-operator created
serviceaccount/prometheus-operator created
deployment.extensions/prometheus-operator created
serviceaccount/prometheus created
```

### 3. Verify that the operators are all running

`kubectl get pods | grep operator`

Expected output:
```
etcd-backup-operator-59cf44997f-qm9x4   1/1     Running   0       99m
etcd-operator-6cb76654cd-497sg          1/1     Running   0       99m
etcd-restore-operator-5dddc644c9-6b2pg  1/1     Running   0       99m
planetscale-operator-6fbfd98864-vr25w   1/1     Running   0       99m
prometheus-operator-78f9dd5bfb-c5xbl    1/1     Running   0       99m
```

### 4. Establish a simple vitess cluster

`kubectl create -f cr_exampledb_keyspace.yaml`

Expected output:
`pscluster.planetscale.com/fresh created`

This will create the following pods:
```
etcd-cluster-fresh-l44tzl2hd5         1/1  Running   0       3m34s
etcd-cluster-fresh-mp4tdct89v         1/1  Running   0       117s
etcd-cluster-fresh-xj824p6b7h         1/1  Running   0       2m37s
prometheus-fresh-example-0            3/3  Running   1       3m32s
proxy-deployment-fresh                1/1  Running   0       3m34s
vtctld-fresh-example-000000000        1/1  Running   1       3m30s
vtgate-fresh-example-000000000        1/1  Running   0       2m2s
vtgate-fresh-example-000000001        1/1  Running   0       2m2s
vttablet-fresh-example-000000001      3/3  Running   1       2m2s
vttablet-fresh-example-000000002      3/3  Running   1       2m2s
vttablet-fresh-example-000001001      3/3  Running   1       2m2s
vttablet-fresh-example-000001002      3/3  Running   1       2m2s
```

### 5. Create the actual database schema and vschema

Use the vtctlclient application to connect and issue vitess commands to vtctld. To enable this, you will need to either [port-forward](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/) to the vtctld pod, or you can [create an externally visible Service](https://kubernetes.io/docs/tasks/access-application-cluster/service-access-application-cluster/) to communicate with vtctld.

Apply the schema and vschema with the commands below, replacing $IP_ADDRESS appropriately, depending on the method chosen above:
```
vtctlclient -server $IP_ADDRESS:15999 ApplySchema -sql "$(cat create_test_table.sql)" messagedb
vtctlclient -server $IP_ADDRESS:15999 ApplyVSchema -vschema "$(cat create_vschema.json)" messagedb
```

### 6. Connect to vtgate using the mysql binary

Exec into the vtgate pod:

`kubectl exec -it vtgate-fresh-example-000000000 bash`

Once you're in the pod, connect directly to vtgate using the MySQL binary:

`mysql -h127.0.0.1 -P3306 -umysql_user --password=mysql_password`

Expected output:
```
vitess@vtgate-fresh-example-000000000:/vt$ mysql -h127.0.0.1 -P3306 -umysql_user --password=mysql_password

Welcome to the MySQL monitor.  Commands end with ; or \g.

Your MySQL connection id is 6

Server version: 5.5.10-Vitess
```
