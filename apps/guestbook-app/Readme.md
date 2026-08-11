This application can be deployed in OpenShift but is intentionally written to not run in OCP out of the box.

OpenShift OpenShift uses things like SCC policies and will not run applications as root by default. The files in this directory will allow you to deploy this application.

## Create project

```
$ oc new-project guestbookphp
```

## Deploy MariaDB using openshift template

Deploy an ephemeral database from the mariadb-ephemeral template

```
$ oc process -n openshift mariadb-ephemeral \
    -p MYSQL_USER=username \
    -p MYSQL_DATABASE=guestbook \
    -p MYSQL_PASSWORD=password \
    -p DATABASE_SERVICE_NAME=mariadb \
    -p NAMESPACE=openshift \
    | oc apply -f -
secret/mariadb created
service/mariadb created
deploymentconfig.apps.openshift.io/mariadb created
```

Now deploy the guestbook configmap

```
$ oc create -f guestbook-configmap.yml
```

Now deploy the app

```
$ oc create -f ocp/guestbook/deployment.yml
```

And finally create the route

```
$ oc expose svc/guestbook-service
$ oc get route
```
