# Lab 5 Deploy an OSB service and attach it to a deployment

Learn to provision a service instance, create a binding for your service instance,
and use that binding to connect your deployment to that service.

# Before we begin

If you skipped Lab0 or otherwise didn't create a Persistant Volume, we need one for Minibroker
to bind the Redis instance to. Find the pv.yaml file in the [Lab0 directory](../Lab0], and run
`kubectl create -f pv.yaml`. Note that while we need to do this because our broker is
provisionin resources inside our Kubernetes cluster, this is not part of the normal
OSB service workflow.

# Provision a Service Instance

Now that we have Minibroker up and running in our Kube cluster, we can use it to provision
a redis instance. If we run `svcat get classes` we can see all the services offered by Minibroker:
```console
$ svcat get classes
     NAME             DESCRIPTION             UUID     
+------------+---------------------------+------------+
  mariadb      Helm Chart for mariadb      mariadb     
  mongodb      Helm Chart for mongodb      mongodb     
  mysql        Helm Chart for mysql        mysql       
  postgresql   Helm Chart for postgresql   postgresql  
  redis        Helm Chart for redis        redis    
```

The same can be done for the Plans of those Classes by running `svcat get plans`. Note that
this response contains a bit more info, so it can display quite a lot of text.
```console
$ svcat get plans
   NAME       CLASS               DESCRIPTION                   UUID        
+---------+------------+--------------------------------+------------------+
.
.
.
  3-2-9     redis        Open source, advanced            redis-3-2-9
                         key-value store. It is
                         often referred to as a data
                         structure server since keys
                         can contain strings, hashes,
                         lists, sets and sorted sets.
  4-0-2     redis        Open source, advanced            redis-4-0-2
                         key-value store. It is
                         often referred to as a data
                         structure server since keys
                         can contain strings, hashes,
                         lists, sets and sorted sets.
  4-0-6     redis        Open source, advanced            redis-4-0-6
                         key-value store. It is
                         often referred to as a data
                         structure server since keys
                         can contain strings, hashes,
                         lists, sets and sorted sets.
  4-0-7     redis        Open source, advanced            redis-4-0-7
                         key-value store. It i
                         often referred to as a data
                         structure server since keys
                         can contain strings, hashes,
                         lists, sets and sorted sets.
  4-0-8     redis        Open source, advanced            redis-4-0-8
                         key-value store. It is
                         often referred to as a data
                         structure server since keys
                         can contain strings, hashes,
                         lists, sets and sorted sets.
  4-0-9     redis        Open source, advanced            redis-4-0-9
                         key-value store. It is
                         often referred to as a data
                         structure server since keys
                         can contain strings, hashes,
                         lists, sets and sorted sets.
```

The numbered tuples are the version of the helm chart minibroker uses to deploy the services we ask it to provison.
For the purposes of this workshop we'll be using the redis Class and the 4-0-9 plan.

Use svcat to provision an instance by running 
```console
$ svcat provisison myredis --class=redis --plan=4-0-9
  Name:        myredis  
  Namespace:   default   
  Status:                
  Class:       redis     
  Plan:        4-0-9
```

If you're interested in a more in-depth explanation of what's going on behind the scenes, check [here](provision.md).
After provisioning, you should be able to list service instances and see your instance:
```console
$ svcat get instances
   NAME     NAMESPACE   CLASS   PLAN    STATUS
+---------+-----------+-------+-------+--------+
  myredis   default     redis   4-0-9   Ready
```

Also, just to make sure it bound to your persistant volume correctly, get all pods and
make sure the new redis pods are up and running. Minibroker provisions them with a randomly
generated name, in this case "jump-albatross".
```console
$ kubectl get pods
NAME                                                  READY     STATUS    RESTARTS   AGE
guestbook-8565b54f47-xbxc7                            1/1       Running   0          7m
jumpy-albatross-redis-master-0                        1/1       Running   0          13m
jumpy-albatross-redis-slave-6d9cbc658f-d6nff          1/1       Running   0          13m
```

# Create a Binding to the Instance

Now we need to create a Binding, which contains the credentials to acces this Service Instance. If you're
interested in what is actually happening behind the scenes during binding, see [here](bind.md). For this
lab, we will use this binding to get the creds into our deployment just like we used the manually
created secret in Lab3. Create a binding by running:
```
svcat bind myredis
```

It will take a few seconds for the pods running the service in the background to spin up. Eventually
svcat should report back the binding as ready:
```console
$ svcat get bindings
   NAME    NAMESPACE   INSTANCE   STATUS
+--------+-----------+----------+--------+
  myredis   default     myredis     Ready
```

Additionally, we should be able to see the Kubernetes secret containing our binding info:
```console
$ kubectl get secrets
NAME                                   TYPE                                  DATA      AGE
bluemix-default-secret                 kubernetes.io/dockercfg               1         1h
bluemix-default-secret-international   kubernetes.io/dockercfg               1         1h
bluemix-default-secret-regional        kubernetes.io/dockercfg               1         1h
default-token-jlszd                    kubernetes.io/service-account-token   3         1h
jumpy-albatross-redis                  Opaque                                1         12m
myredis                                Opaque                                6         9m
```

We can also get the secret and see the creds that have been created for us:
```console
$ kubectl get secrets/myredis -o yaml
apiVersion: v1
data:
  Protocol: cmVkaXM=
  host: anVtcHktYWxiYXRyb3NzLXJlZGlzLW1hc3Rlci5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2Fs
  password: NE1hakQxdTFvQw==
  port: NjM3OQ==
  redis-password: NE1hakQxdTFvQw==
  uri: cmVkaXM6Ly86NE1hakQxdTFvQ0BqdW1weS1hbGJhdHJvc3MtcmVkaXMtbWFzdGVyLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWw6NjM3OQ==
kind: Secret
metadata:
  creationTimestamp: 2018-05-31T22:26:43Z
  name: redis4
  namespace: default
  ownerReferences:
  - apiVersion: servicecatalog.k8s.io/v1beta1
    blockOwnerDeletion: true
    controller: true
    kind: ServiceBinding
    name: redis4
    uid: b21a7e37-6521-11e8-ab01-fe10aa97b9e3
  resourceVersion: "2102"
  selfLink: /api/v1/namespaces/default/secrets/redis4
  uid: b248d18e-6521-11e8-b950-fa2058977f20
type: Opaque
```

Note that you'll have to decode the values out of base64 to see the actual values.

# Attach the Binding to your Deployment

This step will be similar to what we did in Lab3. Run `kubectl edit deployment guestbook` and change the deployment
to pull information from our new secret.
<pre>
apiVersion: apps/v1
kind: Deployment
metadata:
  name: guestbook
  labels:
    app: guestbook
spec:
  replicas: 1
  selector:
    matchLabels:
      app: guestbook
  template:
    metadata:
      labels:
        app: guestbook
    spec:
      containers:
      - name: guestbook
        image: ibmcom/guestbook:v1
        ports:
        - name: http-server
          containerPort: 3000
        env:
        - name: REDIS_MASTER_SERVICE_HOST
          valueFrom:
            secretKeyRef:<b>
              name: myredis </b>
              key: host
        - name: REDIS_MASTER_SERVICE_PORT
          valueFrom:
            secretKeyRef:<b>
              name: myredis</b>
              key: port
        - name: REDIS_MASTER_SERVICE_PASSWORD
          valueFrom:
            secretKeyRef:<b>
              name: myredis</b>
              key: password
</pre>

Note that while we have to manually inject this currently, there is a feature under currently
under development that will automatically inject this information from a binding into a pod or
deployment. 

Now, head back to your web browser and refresh the page. You should see Guestbook with no information
entered. If you click on /info it should display the information for the new Redis instance.

If you add entries into Guestbook, these will be stored in the Redis instance we provisioned using
the service broker. This will be persisted regardless of what we do with our deployment, and we could attach
multiple deployments to the Redis instance using the single Binding we created, or use svcat to create
additional bindings and use those with other deployments. If we had a need for more redis instances, it would
likewise be easy to provision more and bind them in similar ways.
