Title: HAProxy + KeyDB on AKS - take two
Author: Ferdinando Simonetti
Tags: Kubernetes, Kubie, AKS, Redis, Keydb, HAProxy
Category: Kubernetes
Date: 2022-11-23

[Last Article](https://blog.fsimonetti.info/using-a-microsoft-aks-cluster-to-test-keydbs-resilience-with-python-clients) was all about putting quickly KeyDB in use on Kubernetes; hence, hardcoding of KeyDB's **AUTH** password.

Now, however, we'll go a little more *professional*.

## Getting default values for KeyDB's Helm chart

There's a way to specify an *already existing Secret* as password source for KeyDB, and we're going to leverage that opportunity to provide needed AUTH informations to HAProxy.

```
ferdi@DESKTOP-NL6I2OD:~/kind-python-keydb$ helm show values enapter/keydb > keydb-default-values.yml
```

The relevant excerpt of the **values** file is there:

```
password: ""
existingSecret: ""
existingSecretPasswordKey: "password"
```

## Create an appropriate Secret and reinstall the Helm chart using it

The Secret

```
ferdi@DESKTOP-NL6I2OD:~/kind-python-keydb$ kubectl create secret generic keydb-password -n default --dry-run=client -o yaml --from-literal=password="Savignone.2015" > keydb-password.yml

ferdi@DESKTOP-NL6I2OD:~/kind-python-keydb$ cat keydb-password.yml
apiVersion: v1
data:
  password: U2F2aWdub25lLjIwMTU=
kind: Secret
metadata:
  creationTimestamp: null
  name: keydb-password
  namespace: default
```

Referencing it

```
ferdi@DESKTOP-NL6I2OD:~/kind-python-keydb$ kubie ctx keydb

[keydb|default] ferdi@DESKTOP-NL6I2OD:~/kind-python-keydb$ kubectl apply -f keydb-password.yml
secret/keydb-password created

[keydb|default] ferdi@DESKTOP-NL6I2OD:~/kind-python-keydb$ helm list
NAME    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
keydb   default         1               2022-11-21 15:37:08.155778831 +0100 CET deployed        keydb-0.43.1    6.3.1

[keydb|default] ferdi@DESKTOP-NL6I2OD:~/kind-python-keydb$ helm upgrade --install keydb enapter/keydb --set existingSecret=keydb-password
Release "keydb" has been upgraded. Happy Helming!
NAME: keydb
LAST DEPLOYED: Wed Nov 23 09:44:34 2022
NAMESPACE: default
STATUS: deployed
REVISION: 2
TEST SUITE: None
```

Now our KeyDB *direct* public IP is gone, but we can still access it through HAProxy's one.

```
[keydb|default] ferdi@DESKTOP-NL6I2OD:~/kind-python-keydb$ kubectl get svc
NAME              TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                         AGE
haproxy-service   LoadBalancer   10.0.168.218   EDITED   8080:31623/TCP,6379:32570/TCP   15h
keydb             ClusterIP      10.0.137.22    <none>          6379/TCP,9121/TCP               42h
keydb-headless    ClusterIP      None           <none>          6379/TCP                        42h
kubernetes        ClusterIP      10.0.0.1       <none>          443/TCP                         20d
```

## Template Configmap for HAProxy

We're going to define a new ConfigMap like the original one... but with something easily replaceable in place of the original password.

```
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: haproxy-config-template
  namespace: default
data:
  haproxy.cfg: |
    global
    	daemon
    	maxconn 256
    
    defaults
    	mode tcp
    	timeout connect 5000ms
    	timeout client 50000ms
    	timeout server 50000ms
    
    frontend http
    	bind :8080
    	default_backend stats    
    
    backend stats
    	mode http
    	stats enable
    
    	stats enable
    	stats uri /
    	stats refresh 1s
    	stats show-legends
    	stats admin if TRUE
    
    resolvers k8s
      parse-resolv-conf
      hold other           10s
      hold refused         10s
      hold nx              10s
      hold timeout         10s
      hold valid           10s
      hold obsolete        10s
    
    frontend redis-write
        bind *:6379
    	default_backend redis-online
    
    backend redis-online
    	mode tcp
    	balance roundrobin
    	option tcp-check
    	tcp-check send AUTH\ ##REPLACETHIS##\r\n
    	tcp-check expect string +OK
    	tcp-check send PING\r\n
    	tcp-check expect string +PONG
      tcp-check send info\ replication\r\n
    	tcp-check expect string role:active-replica
      server-template keydb 3 server._tcp.keydb-headless.default.svc.cluster.local:6379 check inter 1s resolvers k8s init-addr none
```

After having applied this configuration, we're going to redeploy HAProxy to make use of the template.

```
apiVersion: /v1
kind: Service
metadata:
  name: haproxy-service
  namespace: default
spec:
  type: LoadBalancer
  ports:
    - name: dashboard
      port: 8080
      targetPort: 8080
    - name: redis-write
      port: 6379
      targetPort: 6379
  selector:
    app: haproxy
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: haproxy-deployment
  namespace: default
  labels:
    app: haproxy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: haproxy
  template:
    metadata:
      name: haproxy-pod
      labels:
        app: haproxy
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                    - haproxy
              topologyKey: "kubernetes.io/hostname"
      initContainers:
        - name: configeditor
          image: busybox:1.35.0
          env:
          - name: KEYDBPWD
            valueFrom:
              secretKeyRef:
                name: keydb-password
                key: password
          command:
          - sh
          - '-c'
          - sed "s/##REPLACETHIS##/${KEYDBPWD}/" /tmp/haproxy.cfg > /tmp2/haproxy.cfg  
          volumeMounts:
          - name: config-template
            mountPath: /tmp/haproxy.cfg
            subPath: haproxy.cfg
            readOnly: true
          - name: init-volume
            mountPath: /tmp2
      containers:
        - name: haproxy
          image: haproxy:2.3
          ports:
            - containerPort: 8080
            - containerPort: 6379
            - containerPort: 6380
          volumeMounts:
          - name: init-volume
            mountPath: /usr/local/etc/haproxy/haproxy.cfg
            subPath: haproxy.cfg
            readOnly: true
      restartPolicy: Always
      volumes:
      - name: config-template
        configMap:
          name: haproxy-config-template
      - name: init-volume
        emptyDir: 
```

Basic Kubernetes techniques here:
- setting an environment variable's value from a Secret
- using an initContainer to prepare the environment for the real worker
- using **EmptyDir** volumes

Et voilà!!!