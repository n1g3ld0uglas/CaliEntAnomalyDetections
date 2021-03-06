# Current Anomaly Detection Workflow

References of all anomaly detection jobs can be found here <br/>
https://docs.tigera.io/reference/anomaly-detection/all-detectors 
<br/>

If your cluster does not have applications, you can use the following storefront application:

```
kubectl apply -f https://installer.calicocloud.io/storefront-demo.yaml
```

Create a GlobalAlert of type AnomalyDetection to indicate which anomaly detector to run.
```
kubectl apply -f https://raw.githubusercontent.com/n1g3ld0uglas/CaliEntAnomalyDetections/main/sample-port-scan-globalalert.yaml
```

Verify that a detection cronjob is created for the GlobalAlert:
```
kubectl get cronjobs -n tigera-intrusion-detection -l tigera.io.detector-cycle=detection
```

Verify that a training cronjob is created for the cluster:
```
kubectl get cronjobs -n tigera-intrusion-detection -l tigera.io.detector-cycle=training
```

Introduce the rogue application to test anomaly detection ```IP_Sweep``` alerts:
```
kubectl apply -f https://installer.calicocloud.io/rogue-demo.yaml -n storefront
```

Read logs for the chosen AnomalyDetection pod.
```
kubectl -n tigera-intrusion-detection logs <active-pod>
```

If this fails to generate alerts, run the below command: <br/>
NB: ```Nmap``` is used to discover hosts and services on a computer network by sending packets and analyzing the responses.

```
nmap -Pn -r -p 1-1000 $POD_IP
```

If you don't have this tool installed, run the below install command:
```
sudo apt-get install nmap
```

Alernatively, run this command:
```
nmap -Pn -sS 10.69.0.30 -p 1-2000
```

Create a quarantine security policy into the ```security``` tier:
```
kubectl apply -f https://raw.githubusercontent.com/n1g3ld0uglas/CaliEntAnomalyDetections/main/quarantine.yaml
```

Quarantine the rogue workload by relabelling the pod name:
```
kubectl label pod <name> -n storefront quarantine=true
```

# Old Anomaly Detection Workflow
1.1 Enabling Anomaly Detecion jobs in Calico Enterprise

```
curl https://docs.tigera.io/manifests/threatdef/ad-jobs-deployment.yaml -O
```

1.2 For the managed cluster (ie: Calico Cloud by default):

```
curl https://docs.tigera.io/manifests/threatdef/ad-jobs-deployment-managed.yaml -O
```

Before applying the below YAML file, remember to swap-out my cluster name with the name of your own cluster:

```
        env:
          - name: nigel-gke-cluster
```

You can do this easily with the below command in your CLI:

```
sed -i 's/CLUSTER_NAME/nigel-gke-cluster/g' ad-jobs-deployment.yaml
```

The below YAML manifest enables all known Job ID's currently configurable in Calico Enterprise:
https://docs.tigera.io/v3.7/threat/anomaly-detection/customizing

```
cat << EOF > ad-jobs-deployment.yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: allow-tigera.allow-anomaly-detection
  namespace: tigera-intrusion-detection
spec:
  tier: allow-tigera
  order: 1
  selector: app == 'anomaly-detection'
  types:
  - Ingress
  - Egress
  egress:
  - action: Allow
    protocol: TCP
    destination:
      namespaceSelector: name == 'tigera-elasticsearch'
      selector: elasticsearch.k8s.elastic.co/cluster-name == 'tigera-secure'
      ports:
      - 9200
  - action: Allow
    protocol: UDP
    destination:
      namespaceSelector: projectcalico.org/name == 'kube-system'
      selector: k8s-app == 'kube-dns'
      ports:
      - 53
  - action: Allow
    protocol: UDP
    destination:
      namespaceSelector: projectcalico.org/name == "openshift-dns"
      selector: dns.operator.openshift.io/daemonset-dns == "default"
      ports: [5353]
---
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: allow-tigera.elasticsearch-access-anomaly-detection
  namespace: tigera-elasticsearch
spec:
  tier: allow-tigera
  order: 1
  selector: elasticsearch.k8s.elastic.co/cluster-name == 'tigera-secure'
  types:
  - Ingress
  ingress:
  - action: Allow
    destination:
      ports:
      - 9200
    protocol: TCP
    source:
      namespaceSelector: name == 'tigera-intrusion-detection'
      selector:  app == 'anomaly-detection'
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ad-jobs-deployment
  namespace: tigera-intrusion-detection
  labels:
    app: anomaly-detection
spec:
  replicas: 1
  selector:
    matchLabels:
      app: anomaly-detection
  template:
    metadata:
      labels:
        app: anomaly-detection
    spec:
      imagePullSecrets:
      - name: tigera-pull-secret
      containers:
      - name: ad-jobs
        image: quay.io/tigera/anomaly_detection_jobs:v3.7.0
        resources:
          requests:
            memory: 1G
        securityContext:
          privileged: true
          allowPrivilegeEscalation: true
        env:
          - name: nigel-gke-cluster
            value: "cluster"
          - name: ELASTIC_PORT
            value: "9200"
          - name: AD_max_docs
            value: "2000000"
          - name: AD_train_interval_minutes
            value: "20"
          - name: AD_BytesOutModel_min_size_for_train
            value: "1000"
          - name: AD_SeasonalAD_c
            value: "400"   
          - name: AD_port_scan_threshold
            value: "500"
          - name: AD_ip_sweep_threshold
            value: "32"
          - name: AD_BytesInModel_min_size_for_train
            value: "1000"
          - name: AD_ProcessRestarts_IsolationForest_score_threshold
            value: "0.78"
          - name: AD_ProcessRestarts_threshold
            value: "4"
          - name: AD_DnsLatency_IsolationForest_n_estimators
            value: "100"
          - name: AD_DnsLatency_IsolationForest_score_threshold
            value: "0.836"
          - name: AD_L7Latency_IsolationForest_n_estimators
            value: "100"
          - name: AD_L7Latency_IsolationForest_score_threshold
            value: "-0.836"
          - name: ES_CA_CERT
            value: /certs/es-ca.pem
          - name: ELASTIC_USER
            valueFrom:
              secretKeyRef:
                key: username
                name: tigera-ee-ad-job-elasticsearch-access
          - name: ELASTIC_PASSWORD
            valueFrom:
              secretKeyRef:
                key: password
                name: tigera-ee-ad-job-elasticsearch-access
        volumeMounts:
          - name: host-volume
            mountPath: /home/idsuser/anomaly_detection_jobs/models
          - name: es-certs
            mountPath: /certs/es-ca.pem
            subPath: es-ca.pem
        command: ["python3"]
        args: ["main.py"]
      volumes:
        - name: es-certs
          secret:
            defaultMode: 420
            items:
              - key: tls.crt
                path: es-ca.pem
            secretName: tigera-secure-es-http-certs-public
        - name: host-volume
          hostPath:
                  path: /var/log/calico
EOF                  
```

Run the below command to apply the file:
```
kubectl apply -f ad-jobs-deployment.yaml
```

Find the pod for the running anomaly detection jobs:
```
kubectl get pods -n tigera-intrusion-detection -l app=anomaly-detection
```
Check the logs of the running pod:
```
kubectl logs ad-jobs-deployment-7b45f659f4-v82gj -n tigera-intrusion-detection | grep ERROR
```
