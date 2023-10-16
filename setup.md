# Postgres on GKE setup

## PGO (Postgres Operator) installation

Tutorial [here](https://access.crunchydata.com/documentation/postgres-operator/latest/installation/kustomize)

### Create namespace
Default namespace name is used: `postgres-operator`
Namespace is defined in [namespace.yaml](./kustomize/install/namespace/namespace.yaml)
To create it:

	kubectl apply -k kustomize/install/namespace

### Install PGO

Using single namespace method

	kubectl apply --server-side -k kustomize/install/singlenamespace

To check its status

	kubectl -n postgres-operator get pods

### Cleanup

	kubectl delete -k kustomize/install/singlenamespace
	kubectl delete -k kustomize/install/namespace

## Creating a Postgres cluster

It's not a k8s cluster, but a set of entities within a certain namespace on the cluster you are logged in to.

Tutorial [here](https://access.crunchydata.com/documentation/postgres-operator/latest/tutorials/basic-setup/create-cluster)

To create dev and prod databases run

	kubectl apply -k kustomize/postgres

To delete cluster

	kubectl delete -k kustomize/postgres

## Getting database connection info

To get all info find secret name, by default it's `signal-data-dev-pguser-signal-data-dev` because by default `{clusterName}-pguser-{userName}` when `clusterName` and `userName` are *signal-data-dev*.

	kubectl get secrets -n postgres-operator "signal-data-dev-pguser-signal-data-dev" -o go-template='{{.data}}'

Be aware all fields are base64 encoded.
To get specific info below snipped can be used, saves data in `dbInfo.yaml`

	secretName=signal-data-dev-pguser-signal-data-dev &&
    echo "dbInfo:" > dbInfo.yaml
	echo " dbName: $(kubectl get secrets -n postgres-operator "${secretName}" -o go-template='{{.data.dbname | base64decode}}')" >> dbInfo.yaml &&
    echo " user: $(kubectl get secrets -n postgres-operator "${secretName}" -o go-template='{{.data.user | base64decode}}')" >> dbInfo.yaml &&
    echo " password: $(kubectl get secrets -n postgres-operator "${secretName}" -o go-template='{{.data.password | base64decode}}')" >> dbInfo.yaml

### GKE tunnel

Tunnel to cluster to connect to DB. Currently it's broken [github issue](https://github.com/kubernetes/kubectl/issues/1169)


	PG_CLUSTER_PRIMARY_POD=$(kubectl get pod -n postgres-operator -o name -l postgres-operator.crunchydata.com/cluster=signal-data-dev,postgres-operator.crunchydata.com/role=master)
	kubectl -n postgres-operator port-forward "${PG_CLUSTER_PRIMARY_POD}" 5432:5432

### GKE pod interactive mode

Run pod in interactive to connect to DB.

	PG_CLUSTER_PRIMARY_POD=$(kubectl get pod -n postgres-operator -o name -l postgres-operator.crunchydata.com/cluster=signal-data-dev,postgres-operator.crunchydata.com/role=master)
	kubectl exec -it -n postgres-operator "${PG_CLUSTER_PRIMARY_POD}" -- psql

## Configuring databases (DB clusters)

Config is saved in k8s files. [init-sql-config.yaml](./kustomize/postgres/init-sql-config.yaml) is applied when creating a new database.

[Spec doc](https://access.crunchydata.com/documentation/postgres-operator/latest/references/crd/postgrescluster)
