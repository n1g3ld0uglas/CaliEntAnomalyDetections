# CaliEntAnomalyDetections
Enabling Anomaly Detecion jobs in Calico Enterprise

Before applying the below YAML file, remember to swap-out my cluster name with the name of your own cluster:

```
        env:
          - name: nigel-gke-cluster
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
          # The job looks for pods in your cluster that are sending packets to one destination on multiple ports.
          # This may indicate an attacker has gained control of a pod and is gathering reconnaissance on what else they can reach.
          # The job compares pods both with other pods in their replica set, and with other pods in the cluster generally.     
          - name: AD_port_scan_threshold
            value: "500"
          # It is a threshold for triggering an anomaly for the ip_sweep job
          # This is a number of unique destination IPs called from the specific source_name_aggr in the same source_namespace, and the same bucket.
          - name: AD_ip_sweep_threshold
            value: "32"
          #   
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



