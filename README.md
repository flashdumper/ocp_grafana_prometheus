# Replacing Grafana in OCP 3.11


This guide shows how to install a new Grafana (from Docker Hub) with RW permissions and integrate it with existing Prometheus on OpenShift Cluster.

Though OCP 3.11 is being used in this guide, it's will work with other OCP versions. 


## Overview
1. Pulling and deploying the latest Grafana image from Docker Hub.
3. Configuring persistent storage for Grafana.
2. Changing Prometheus StatefulSet to allow new Grafana. 
4. Integrating Grafana with existing Prometheus
5. Grafana Additional configuration




## Pulling the image

We are using standard Grafana image from the upsteam on Docker Hub for this task using OpenShift CLI. You can use web console if you'd like to. 
First, switch to `openshift-monitoring` project and pull the latest image.

```
# oc project openshift-monitoring
# oc new-app --docker-image=docker.io/grafana/grafana --name=grafana54
```

If everything goes normally, you should have several resource created including Grafana pod and services.

```
# oc get all -l app=grafana54

NAME                    READY     STATUS    RESTARTS   AGE
pod/grafana54-1-h4bb9   1/1       Running   0          35m

NAME                                DESIRED   CURRENT   READY     AGE
replicationcontroller/grafana54-1   1         1         1         35m

NAME                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/grafana54   ClusterIP   100.65.159.152   <none>        3000/TCP   35m

NAME                                           REVISION   DESIRED   CURRENT   TRIGGERED BY
deploymentconfig.apps.openshift.io/grafana54   1          1         1         config,image(grafana54:latest)

NAME                                       DOCKER REPO                                                       TAGS      UPDATED
imagestream.image.openshift.io/grafana54   docker-registry.default.svc:5000/openshift-monitoring/grafana54   latest    35 minutes ago
```

Expose Grafana service externally


```
# oc expose service/grafana54

# oc get route -l app=grafana54
NAME        HOST/PORT                                               PATH      SERVICES    PORT       TERMINATION   WILDCARD
grafana54   grafana54-openshift-monitoring.apps.ups.icc.dzuev.pro             grafana54   3000-tcp                 None
```

Finally, verify the connectivity from CLI and your browser.

```
# curl grafana54-openshift-monitoring.apps.ups.icc.dzuev.pro
<a href="/login">Found</a>.
```

You can use HTTPS instead of HTTP if needed. 


## Grafana Persistent storage 

Depending whether you use storage-classes for dynamic storage provisioning, you might need to change `storage-class` annotation, or create a pv manually.

Create a PVC for Grafana.

```
# oc create -f grfana54pvc.yml
# cat grfana54pvc.yml

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
    volume.beta.kubernetes.io/storage-class: glusterfs-storage-infra
    volume.beta.kubernetes.io/storage-provisioner: kubernetes.io/glusterfs
  name: grafana54
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
```


In standard grafana there are two main directories we need to create and attach persistent volume to: 
- /var/lib/grafana - Grafana TSDB location
- /etc/grafana - grafana configuration files

Edit Grafana DeploymentConfig and add persistent storage configuration for `/var/lib/grafana` first.

```
# oc edit dc grafana54
spec:  
  ...
  template:
    ...
    spec:
      volumes:
      - name: grafana54-volume
        persistentVolumeClaim:
          claimName: grafana54
      containers:
        name: grafana54
        volumeMounts:
        - mountPath: /var/lib/grafana
          name: grafana54-volume
      ...
```

Copy files from /etc/grafana to /var/lib/grafana

```
oc rsh grafana54-4-g6w9t bash -c 'cp -R /etc/grafana/* /var/lib/grafana/'
```

Edit grafana54 DC one more time and add the second folder `/etc/grafana` to persistent volume.

```
# oc edit dc grafana54
...
        volumeMounts:
        - mountPath: /var/lib/grafana
          name: grafana54-volume
        - mountPath: /etc/grafana
          name: grafana54-volume
...
```


## Connecting to Prometheus 

We need to adjust Prometheus StatefulSet in order to Grafana to connect to it. I was not able to figure out how to integrate standard Grafana to work with Prometheus using oauth. So the quick workaround was to make prometheus to listen on all the interfaces.

Edit prometheus sts.

```
# oc edit sts prometheus-k8s
```

search for `prometheus:v3` and replace `--web.listen-address=localhost:9090`  with `--web.listen-address=:9090` like in the example below.

```
      containers:
      - args:
        - --web.console.templates=/etc/prometheus/consoles
        - --web.console.libraries=/etc/prometheus/console_libraries
        - --config.file=/etc/prometheus/config_out/prometheus.env.yaml
        - --storage.tsdb.path=/prometheus
        - --storage.tsdb.retention=15d
        - --web.enable-lifecycle
        - --storage.tsdb.no-lockfile
        - --web.external-url=https://prometheus-k8s-openshift-monitoring.apps.ups.icc.dzuev.pro/
        - --web.route-prefix=/
        - --web.listen-address=:9090
        image: nexus.icc.dzuev.pro:8083/openshift3/prometheus:v3.11.43
```

That should trigger prometheus to restart the deployment. Check if you have access to prometheus.

```
curl http://prometheus-operated.openshift-monitoring.svc:9090
<a href="/graph">Found</a>.
```

Perfect, now login to grafana via browser using standard username and password `admin:admin`.
Then change the password when asked on the following screen and press save.

Go to `Data Sources` -> `Add Data Source` -> `Prometheus`.

In URL use `http://prometheus-operated.openshift-monitoring.svc:9090` and press `Save and Test`.



## Grafana additional configuration



