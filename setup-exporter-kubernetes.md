## Setup Prometheus Node Exporter on Kubernetes

### **Introduction**

In a dynamic and scalable environment like Kubernetes, comprehensive monitoring of your cluster's underlying nodes is paramount. While Kubernetes provides basic health checks, deeper insights into hardware and operating system metrics are essential for performance analysis, capacity planning, and troubleshooting. Prometheus is the de-facto standard for monitoring in Kubernetes, and the Node Exporter is its dedicated agent for collecting host-level metrics.

This project will guide you through deploying Prometheus Node Exporter as a `DaemonSet` across your Kubernetes cluster nodes. By doing so, you'll enable your Prometheus instance (already running within the cluster) to scrape and visualize critical metrics from every node, giving you unparalleled visibility into your cluster's infrastructure.

### **Objectives**

Upon successful completion of this project, you will be able to:

  * Understand the fundamental purpose and functionality of Prometheus Node Exporter within a Kubernetes context.
  * Deploy Node Exporter as a `DaemonSet` to ensure it runs on every node in your Kubernetes cluster.
  * Configure your existing Prometheus instance within Kubernetes to automatically discover and scrape metrics from the deployed Node Exporter instances.
  * Access the Prometheus UI to verify that metrics are being collected successfully.
  * Explore and analyze various hardware and operating system metrics provided by Node Exporter using PromQL.

### **Prerequisites**

To successfully complete this project, you will need:

  * **Working Kubernetes Cluster:** A functional Kubernetes cluster (e.g., Minikube, Kind, Google Kubernetes Engine (GKE), Amazon Elastic Kubernetes Service (EKS), Azure Kubernetes Service (AKS)).
  * **Kubernetes CLI (`kubectl`):** `kubectl` installed and configured on your local machine to interact with your Kubernetes cluster.
  * **Basic Prometheus Installation:** A basic Prometheus setup already running within your Kubernetes cluster. This project assumes Prometheus is installed (e.g., via Helm chart or manual deployment) and its configuration can be modified. Typically, Prometheus is deployed in a dedicated namespace, often `monitoring`.
  * **Tools:** A text editor to create and modify YAML files.

### **Estimated Time**

2-4 hours

### **Tasks Outline**

1.  **Understand How Node Exporter Works** in a Kubernetes environment.
2.  **Deploy Node Exporter as a DaemonSet** in your Kubernetes cluster.
3.  **Create a Kubernetes Service** for Node Exporter (for discovery).
4.  **Configure Prometheus** to scrape metrics from Node Exporter using Kubernetes service discovery.
5.  **Verify the metrics** in the Prometheus UI.
6.  **Explore** the metrics provided by Node Exporter.

-----

### **Project Tasks: Detailed Steps**

Follow these detailed steps on your local machine with `kubectl` configured for your Kubernetes cluster.

#### **Task 1 - Understand How Node Exporter Works in Kubernetes**

Node Exporter is a lightweight application designed to run directly on a host and expose metrics about that host. In a Kubernetes context:

  * **DaemonSet:** Node Exporter is typically deployed as a `DaemonSet`. This Kubernetes controller ensures that a replica of the Node Exporter pod runs on *every* node in your cluster. This guarantees that metrics are collected from all worker nodes.
  * **Containerized:** It runs as a containerized application within each pod, collecting data from the host's `/proc` and `/sys` filesystems (which need to be mounted into the container).
  * **Key Metrics:** Node Exporter exposes metrics about:
      * CPU and memory usage
      * Disk I/O and filesystem usage
      * Network statistics (bytes received/transmitted, errors)
      * System load averages
      * Running processes
      * And many more host-level statistics.
  * **Prometheus Discovery:** Prometheus uses Kubernetes service discovery mechanisms (e.g., `kubernetes_sd_configs` with `role: endpoints`) to find Node Exporter pods and automatically scrape metrics from them.

#### **Task 2 - Deploy Node Exporter as a DaemonSet**

First, ensure you have a `monitoring` namespace (or use your existing Prometheus namespace).

```bash
kubectl create namespace monitoring
```

Now, create the YAML definition for the Node Exporter DaemonSet and a corresponding Service.

1.  **Create a YAML file for the Node Exporter DaemonSet:**
    Create a file named `node-exporter-daemonset.yaml`.

    ```yaml
    # node-exporter-daemonset.yaml
    apiVersion: apps/v1
    kind: DaemonSet
    metadata:
      name: node-exporter
      namespace: monitoring # Deploy in the monitoring namespace
      labels:
        app.kubernetes.io/name: node-exporter
        app.kubernetes.io/component: metrics
    spec:
      selector:
        matchLabels:
          app.kubernetes.io/name: node-exporter
      template:
        metadata:
          labels:
            app.kubernetes.io/name: node-exporter
            app.kubernetes.io/component: metrics
        spec:
          # Required for Node Exporter to access host filesystems for metrics
          hostPID: true
          hostIPC: true
          hostNetwork: true # Important for capturing host-level network metrics without NAT
          dnsPolicy: ClusterFirstWithHostNet # Required with hostNetwork

          containers:
            - name: node-exporter
              image: prom/node-exporter:latest # Using latest for simplicity, pin to specific version for production
              ports:
                - containerPort: 9100
                  name: metrics
                  hostPort: 9100 # Expose on host's port 9100 (required with hostNetwork)
              args:
                - "--path.procfs=/host/proc" # Mount host /proc
                - "--path.sysfs=/host/sys"   # Mount host /sys
                - "--path.rootfs=/host/root" # Mount host /
                - "--collector.filesystem.mount-points-exclude=^/(dev|proc|sys|var/lib/docker/.+|var/lib/kubelet/pods/.+)($|/)"
              resources:
                limits:
                  memory: "100Mi"
                  cpu: "100m"
                requests:
                  memory: "50Mi"
                  cpu: "50m"
              securityContext:
                # Node Exporter generally runs as non-root, but needs access to host namespaces.
                # hostPID, hostIPC, hostNetwork bypass some of the isolation for host-level metrics.
                # Some deployments might use readOnlyRootFilesystem for stronger security.
                runAsNonRoot: true
                runAsUser: 65534 # nobody user
                allowPrivilegeEscalation: false
                capabilities:
                  drop:
                    - ALL
                  add:
                    - SYS_RAWIO # Needed for some collectors like diskstats
              volumeMounts:
                - name: proc
                  mountPath: /host/proc
                  readOnly: true
                - name: sys
                  mountPath: /host/sys
                  readOnly: true
                - name: root
                  mountPath: /host/root
                  readOnly: true
          volumes:
            - name: proc
              hostPath:
                path: /proc
            - name: sys
              hostPath:
                path: /sys
            - name: root
              hostPath:
                path: /
    ```

      * **`hostPID: true`, `hostIPC: true`, `hostNetwork: true`**: These are crucial for Node Exporter to collect accurate host-level metrics by breaking out of the container's isolated network/PID/IPC namespaces.
      * **`hostPort: 9100`**: Exposes port 9100 directly on the host, making it easily discoverable by Prometheus.
      * **`args`**: Explicitly tells Node Exporter where to find the mounted host filesystems.
      * **`securityContext`**: Ensures the container runs as a non-root user (`nobody`).
      * **`volumeMounts` / `volumes`**: Mounts the host's `/proc`, `/sys`, and root (`/`) filesystems into the container as read-only.

2.  **Create a YAML file for the Node Exporter Service:**
    Although Node Exporter often uses `hostPort` and `hostNetwork`, a `Service` is still useful for Prometheus's Kubernetes service discovery (which typically targets Service endpoints) and for logical grouping. Create a file named `node-exporter-service.yaml`.

    ```yaml
    # node-exporter-service.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: node-exporter # Service name Prometheus will discover
      namespace: monitoring
      labels:
        app.kubernetes.io/name: node-exporter
        app.kubernetes.io/component: metrics
    spec:
      selector:
        app.kubernetes.io/name: node-exporter # Select pods with this label
      ports:
        - name: metrics
          port: 9100      # Service port
          targetPort: 9100 # Pod port
          protocol: TCP
      type: ClusterIP # Internal cluster access
    ```

      * **`selector`**: Matches the labels of the `node-exporter` pods created by the DaemonSet.
      * **`port: 9100`**: The port that the Service exposes *internally* within the cluster.

3.  **Apply the YAML files using `kubectl`:**

    ```bash
    kubectl apply -f node-exporter-daemonset.yaml
    kubectl apply -f node-exporter-service.yaml
    ```

4.  **Verify the deployment:**
    Check that the DaemonSet is running and that pods are deployed on your nodes.

    ```bash
    kubectl get daemonset -n monitoring
    # Expected output: Shows node-exporter with DESIRED and CURRENT matching, and READY count.

    kubectl get pods -l app.kubernetes.io/name=node-exporter -n monitoring
    # Expected output: Shows node-exporter pods, one per node, all in Running state.

    kubectl get svc -n monitoring node-exporter
    # Expected output: Shows the node-exporter service with ClusterIP.
    ```

5.  **Verify direct access to metrics (optional):**
    Pick one of the `node-exporter` pod names (e.g., `node-exporter-abcde`) from `kubectl get pods`.

    ```bash
    kubectl port-forward -n monitoring <node-exporter-pod-name> 9100:9100
    ```

    Then, open your web browser or use `curl` on your local machine: `http://localhost:9100/metrics`. You should see the raw metrics. Press `Ctrl+C` in your terminal to stop port-forwarding.

#### **Task 3 - Configure Prometheus to Scrape Metrics from Node Exporter**

Now, you need to update your Prometheus configuration (`ConfigMap`) to tell it how to discover and scrape the Node Exporter metrics.

1.  **Identify your Prometheus ConfigMap:**
    If you installed Prometheus via Helm, its ConfigMap name might be something like `prometheus-server-conf` or `prometheus-<release_name>-server`. You can find it with:

    ```bash
    kubectl get configmaps -n monitoring | grep prometheus
    ```

    Let's assume your Prometheus server's configuration is in a ConfigMap named `prometheus-server-conf` in the `monitoring` namespace.

2.  **Edit the Prometheus configuration ConfigMap:**

    ```bash
    kubectl edit configmap prometheus-server-conf -n monitoring
    ```

      * Look for the `data:` section, and within it, the `prometheus.yml:` key. The value of this key is a multi-line string representing your Prometheus configuration.

3.  **Add a new scrape job for Node Exporter:**
    Under the `scrape_configs:` section in the `prometheus.yml` content, add the following job. Ensure correct YAML indentation.

    ```yaml
    # ... other scrape_configs ...
    - job_name: 'node-exporter'
      kubernetes_sd_configs: # Use Kubernetes service discovery
        - role: endpoints   # Discover services by their endpoints
          namespaces:
            names: ['monitoring'] # Look only in the 'monitoring' namespace
      relabel_configs:
        # Keep only targets that are part of the 'node-exporter' service
        - source_labels: [__meta_kubernetes_service_name]
          regex: node-exporter
          action: keep
        # Relabel __address__ to use the host IP if hostNetwork is true (optional but good practice)
        # Note: If hostNetwork: true and hostPort: 9100 are used, the IP will already be the node's IP.
        # This relabel ensures the instance label shows the node name.
        - source_labels: [__meta_kubernetes_pod_node_name]
          target_label: instance
          action: replace
        # You might also want to set a custom __address__ if hostNetwork isn't used
        # - source_labels: [__address__]
        #   regex: (.*):9100
        #   target_label: __address__
        #   replacement: "${1}:9100" # Use the pod IP with port 9100

      # This config can also be simplified if you are okay with scraping all endpoints
      # and then filtering by port or other labels.
    ```

      * **`kubernetes_sd_configs`**: This tells Prometheus to use Kubernetes' API to discover targets.
      * **`role: endpoints`**: Prometheus will look at Service Endpoints.
      * **`namespaces: ['monitoring']`**: Limits discovery to the `monitoring` namespace.
      * **`relabel_configs`**: This is crucial for filtering and relabeling discovered targets.
          * The first `relabel_configs` ensures that only endpoints associated with the `node-exporter` service (identified by `__meta_kubernetes_service_name`) are kept.
          * The second `relabel_configs` renames the `instance` label from the pod IP to the actual node name, which is more readable.

4.  **Save the ConfigMap.** (Saving automatically updates it.)

5.  **Restart the Prometheus deployment to load the new configuration:**
    Changes to a ConfigMap are not automatically picked up by running pods. You need to trigger a rolling restart of your Prometheus deployment.
    Find your Prometheus deployment name:

    ```bash
    kubectl get deployments -n monitoring | grep prometheus
    # Example output: prometheus-server
    ```

    Then, trigger the restart:

    ```bash
    kubectl rollout restart deployment prometheus-server -n monitoring
    # Replace 'prometheus-server' with your actual Prometheus deployment name
    ```

    Wait a few moments for the old pods to terminate and new ones to start. You can monitor with `kubectl get pods -n monitoring -w`.

#### **Task 4 - Verify Metrics in Prometheus**

Now, let's confirm that Prometheus is collecting metrics from your Node Exporter instances.

1.  **Access the Prometheus UI:**
    You may need to port-forward the Prometheus service to access its UI from your local machine.
    Find your Prometheus service name:

    ```bash
    kubectl get svc -n monitoring | grep prometheus
    # Example output: prometheus-server
    ```

    Then, port-forward:

    ```bash
    kubectl port-forward svc/prometheus-server 9090:9090 -n monitoring
    # Replace 'prometheus-server' with your actual Prometheus service name
    ```

    Open your web browser and navigate to `http://localhost:9090`.

2.  **Check the "Targets" page:**
    In the Prometheus UI, go to **Status** -\> **Targets**. You should see the `node-exporter` job listed, and each target (representing a node where Node Exporter is running) should have a state of **UP**. This confirms Prometheus is successfully connecting and scraping metrics.

3.  **Run a query to view Node Exporter metrics:**
    Go to the **Graph** tab in Prometheus. In the expression input box, type a common Node Exporter metric name, for example:

    ```
    node_cpu_seconds_total
    ```

    Click the **Execute** button. You should see time series data related to CPU usage from all your cluster nodes. You can use the `instance` label to filter or identify specific nodes.

#### **Task 5 - Explore Metrics Provided by Node Exporter**

Node Exporter provides a comprehensive set of metrics. Use the Prometheus query interface to explore some key ones:

  * **Available Memory (bytes):** `node_memory_MemAvailable_bytes`
  * **Total CPU seconds:** `node_cpu_seconds_total` (use `rate` for per-second changes)
  * **Load Average (1-minute):** `node_load1`
  * **Filesystem Free Space (bytes):** ` node_filesystem_avail_bytes{mountpoint="/"}  ` (adjust `mountpoint` as needed)
  * **Network Bytes Received (total):** `node_network_receive_bytes_total`
  * **Number of Running Processes:** `node_procs_running`
  * **Context Switches:** `node_context_switches_total`

**Use Prometheus expressions (PromQL) to analyze data:**

  * **CPU Usage Percentage (averaged over 5 minutes, excluding idle):**
    ```promql
    100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
    ```
  * **Disk I/O Read Operations per second (over 1 minute):**
    ```promql
    rate(node_disk_reads_completed_total[1m])
    ```
  * **Network Throughput in MB/s (over 5 minutes):**
    ```promql
    (rate(node_network_receive_bytes_total[5m]) + rate(node_network_transmit_bytes_total[5m])) / (1024 * 1024)
    ```

### **Conclusion**

By completing this project, you have successfully set up Prometheus Node Exporter as a `DaemonSet` in your Kubernetes cluster, enabling comprehensive monitoring of your node-level metrics. You've integrated Node Exporter with your existing Prometheus instance, learned to configure Kubernetes service discovery, and explored the wealth of data it provides through PromQL queries.

This setup is a fundamental component of a robust Kubernetes monitoring strategy. You can now extend this by:

  * **Integrating Grafana:** To create rich, customizable dashboards for visualizing these metrics alongside your cluster's overall health.
  * **Configuring Alertmanager:** To define alert rules for critical node performance issues (e.g., high CPU, low memory, full disk) and receive notifications.
  * **Adding other Exporters:** To collect metrics from specific applications or services running within your pods.
  * **Exploring advanced PromQL:** To create more complex queries for in-depth analysis and custom metrics.

-----

### **Important Notes and Troubleshooting**

  * **RBAC (Role-Based Access Control):** For production environments, it's a best practice to define a dedicated `ServiceAccount`, `ClusterRole`, and `ClusterRoleBinding` for the Node Exporter `DaemonSet`. While the provided YAML might work for basic metrics in some clusters due to default permissions, explicit RBAC ensures the Node Exporter has only the necessary permissions to access host resources without elevating privileges unnecessarily. For example, some collectors (like `textfile`) might require specific host paths or permissions that default `nobody` user won't have.
  * **`hostNetwork: true` and `hostPort: 9100`**: These settings are common for Node Exporter as they allow it to access network interfaces and bind to a port on the actual host, providing more accurate host-level network metrics and simplified discovery. Without `hostNetwork: true`, network metrics would be from the pod's perspective, not the host's.
  * **YAML Indentation:** YAML is sensitive to indentation. Ensure your copy-pasted configurations maintain proper spacing. Incorrect indentation will lead to Kubernetes API server errors when applying or `promtool` errors when validating.
  * **Prometheus ConfigMap Update:** Remember that changes to a ConfigMap are *not* immediately reflected by running pods. You must perform a rolling restart of your Prometheus deployment for the new configuration to take effect.
  * **Firewalls:** While Kubernetes networking generally handles traffic within the cluster, ensure that any external firewalls or cloud security groups (e.g., AWS Security Groups, Azure Network Security Groups) allow traffic on port 9100 from your Prometheus server's IP address if Prometheus is outside the cluster, or if you ever try to access Node Exporter metrics directly from outside the cluster.
  * **Prometheus Service Discovery Debugging:** If targets don't show up as `UP` in Prometheus UI's "Targets" page:
      * Check Node Exporter pod logs (`kubectl logs -n monitoring <node-exporter-pod-name>`).
      * Verify Node Exporter is running on port 9100 by `port-forwarding` to a pod and `curl`ing.
      * Check Prometheus server logs (`kubectl logs -n monitoring <prometheus-pod-name>`) for scrape errors.
      * Double-check your `relabel_configs` in `prometheus.yml` – a common source of targets being dropped.
