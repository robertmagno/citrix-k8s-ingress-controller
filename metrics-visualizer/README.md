Metrics Visualization of NetScaler Appliances in Kubernetes
===

This document describes how the [NetScaler Metrics Exporter](https://github.com/citrix/netscaler-metrics-exporter) and [Prometheus-Operator](https://github.com/coreos/prometheus-operator) can be used to auto-detect and monitor VPX/CPX ingress devices and CPX-EW (east-west) devices. Moitoring dashboards setup for detected devices will be brought up as Grafana dashboard.


Launching Promethus-Operator
---
Prometheus Operator has an expansive method of monitoring services on Kubernetes. It makes use ServiceMonitors defined by CRDs to automatically detect services and their corresponding pod endpoints. To get started a basic working model, this guide makes use of [kube-prometheus](https://github.com/coreos/prometheus-operator/tree/master/contrib/kube-prometheus) and its [manifest](https://github.com/coreos/prometheus-operator/tree/master/contrib/kube-prometheus/manifests) files.
```
git clone https://github.com/coreos/prometheus-operator.git
kubectl create -f prometheus-operator/contrib/kube-prometheus/manifests/
```
This creates several pods and services, of which ```prometheus-k8s-xx``` pods aggregate and timestamp metrics collected from NetScaler devices, and the ```grafana``` pod can be used for visualization. An output similar to this should be seen:
```
$ kubectl get pods -n monitoring
NAME                                   READY     STATUS    RESTARTS   AGE
alertmanager-main-0                    2/2       Running   0          2h
alertmanager-main-1                    2/2       Running   0          2h
alertmanager-main-2                    2/2       Running   0          2h
grafana-5b68464b84-5fvxq               1/1       Running   0          2h
kube-state-metrics-6588b6b755-d6ftg    4/4       Running   0          2h
node-exporter-4hbcp                    2/2       Running   0          2h
node-exporter-kn9dg                    2/2       Running   0          2h
node-exporter-tpxhp                    2/2       Running   0          2h
prometheus-k8s-0                       3/3       Running   1          2h
prometheus-k8s-1                       3/3       Running   1          2h
prometheus-operator-7d9fd546c4-m8t7v   1/1       Running   0          2h
```
**NOTE:** It may be preferable to expose the Prometheus and Grafana pods via NodePorts. To do so, the prometheus-service.yaml and grafana-service.yaml files will need to be modified as follows:


<details>
<summary>prometheus-service.yaml</summary>
<br>

```
apiVersion: v1
kind: Service
metadata:
  labels:
    prometheus: k8s
  name: prometheus-k8s
  namespace: monitoring
spec:
  type: NodePort
  ports:
  - name: web
    port: 9090
    targetPort: web
  selector:
    app: prometheus
    prometheus: k8s
```

</details>


<details>
<summary>grafana-service.yaml</summary>
<br>

```
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
spec:
  type: NodePort
  ports:
  - name: http
    port: 3000
    targetPort: http
  selector:
    app: grafana
```

</details>


To apply these changes into the kubernetes cluseter run:
```
kubectl apply -f prometheus-service.yaml
kubectl apply -f grafana-service.yaml
```

Configuring NetScaler Metrics Exporter
---
This section describes how to integrate the NetScaler Metrics Exporter with the VPX/CPX ingress or CPX-EW devices. 

<details>
<summary>VPX Ingress Device</summary>
<br>

To monitor an ingress VPX device, the netscaler-metrics-exporter will be run as a pod within the kubernetes cluster. The IP of the VPX ingress device will be provided as an argument to the exporter. An example yaml file to deploy such an exporter is given below:

```
apiVersion: v1
kind: Pod
metadata:
  name: exporter-vpx-ingress
  labels:
    app: exporter-vpx-ingress
spec:
  containers:
    - name: exporter
      image: "quay.io/citrix/netscaler-metrics-exporter:latest"
            imagePullPolicy: Always
      args:
        - "--target-nsip=<IP_and_port_of_VPX>"
        - "--port=8888"
---
kind: Service
apiVersion: v1
metadata:
  name: exporter-vpx-ingress
  labels:
    service-type: citrix-adc-monitor
spec:
  selector:
    name: exporter-vpx-ingress
  ports:
    - name: exporter-port
      port: 8888
      targetPort: 8888
```
The IP and port of the VPX device needs to be filled in as the ```--target-nsip``` (Eg. ```--target-nsip=10.0.0.20```). 
</details>

<details>
<summary>CPX Ingress Device</summary>
<br>
  
To monitor a CPX ingress device, the exporter is added as a side-car. An example yaml file of a CPX ingress device with an exporter as a side car is given below;
```
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: cpx-ingress
  labels:
    app: cpx-ingress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cpx-ingress
  template:
    metadata:
      labels:
        app: cpx-ingress
      annotations:
        NETSCALER_AS_APP: "True"
    spec:
      serviceAccountName: cpx
      containers:
        # Adding exporter as a side-car
        - name: exporter
          image: "quay.io/citrix/netscaler-metrics-exporter:latest"
          imagePullPolicy: Always
          args:
            - "--target-nsip=127.0.0.1"
            - "--port=8888"
        - name: cpx-ingress
          image: "us.gcr.io/citrix-217108/citrix-k8s-cpx-ingress:latest"
          imagePullPolicy: Always
          securityContext:
            privileged: true
          env:
            - name: "EULA"
              value: "YES"
            - name: "NS_PROTOCOL"
              value: "HTTP"
            #Define the NITRO port here
            - name: "NS_PORT"
              value: "9080"
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
            - name: nitro-http
              containerPort: 9080
---
kind: Service
apiVersion: v1
metadata:
  name: exporter-cpx-ingress
  labels:
    service-type: citrix-adc-monitor
spec:
  selector:
    app: cpx-ingress
  ports:
    - name: exporter-port
      port: 8888
      targetPort: 8888
```
Here, the exporter uses the ```127.0.0.1``` local IP to fetch metrics from the CPX.

</details>


<details>
<summary>CPX-EW Device</summary>
<br>

To monitor a CPX-EW (east-west) device, the exporter is added as a side-car. An example yaml file of a CPX-EW device with an exporter as a side car is given below;
```
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: cpx-ew
spec:
  template:
    metadata:
      name: cpx-ew
      labels:
        app: cpx-ew
      annotations:
        NETSCALER_AS_APP: "True"
    spec:
      serviceAccountName: cpx
      hostNetwork: true
      containers:
        - name: cpx
          image: "in-docker-reg.eng.citrite.net/cpx-dev/cpx:12.1-48.118"
          securityContext: 
             privileged: true
          env:
          - name: "EULA"
            value: "yes"
          - name: "NS_NETMODE"
            value: "HOST"
          #- name: "kubernetes_url"
          #  value: "https://10..xx.xx:6443"
        # Add exporter as a sidecar
        - name: exporter
          image: "quay.io/citrix/netscaler-metrics-exporter:latest"
          args:
            - "--target-nsip=192.168.0.2:80"
            - "--port=8888"
          imagePullPolicy: Always
---
kind: Service
apiVersion: v1
metadata:
  name: exporter-cpx-ew
  labels:
    service-type: citrix-adc-monitor
spec:
  selector:
    app: cpx-ew
  ports:
    - name: exporter-port
      port: 8888
      targetPort: 8888
```
Here, the exporter uses the ```192.168.0.2``` local IP to fetch metrics from the CPX.

</details>



ServiceMonitors to Detect NetScalers
---
Till this point, the NetScaler Metrics Exporters was setup to collect data from the VPX/CPX ingress and CPX-EW devices. Now, these Exporters needs to be detected by Prometheus Operator so that the metrics which are collected can be timestamped, stored, and exposed for visualization on Grafana. Prometheus Operator uses the concept of ```ServiceMonitors``` to detect pods belonging to a service, using the labels attached to that service. 

The following example file will detect all the Exporter endpoints associated with the Exporter services which have the label ```service-type: citrix-adc-monitor``` associated with them.
```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: citrix-adc-servicemonitor
  labels:
    servicemonitor: citrix-adc
spec:
  endpoints:
  - interval: 30s
    port: exporter-port
  selector:
    matchLabels:
      service-type: citrix-adc-monitor
  namespaceSelector:
    matchNames:
    - monitoring
    - default
```


Visualization of Metrics
---
The NetScaler instances which were detected for monitoring will appear in the ```Targets``` page of the prometheus container. It canbe accessed using ```http://<k8s_cluster_ip>:<prometheus_nodeport>/targets``` and will look similar to the screenshot below


![image](./images/prometheus-targets.png)

To view the metrics graphically,
1. Log into grafana using ```http://<k8s_cluster_ip>:<grafafa_nodeport>``` with default credentials ```admin:admin```

2. Import the [sample grafana dashboard](https://github.com/citrix/netscaler-metrics-exporter/blob/master/sample_grafana_dashboard.json) by selecting the ```+``` icon on the left panel and clicking import.

<img src="./images/grafana-import-json.png" width="200">


3. A dashboard containing graphs similar to the following should appear

![image](./images/grafana-dashboard.png)

4. The dashboard can be further enhanced using Grafana's [documentation](http://docs.grafana.org/) or demo [videos](https://www.youtube.com/watch?v=mgcJPREl3CU).
