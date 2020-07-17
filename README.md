# KubeCon EU 2020

This repo contains the source material for my demos for KubeCon EU 2020.

## Simple Automated Thanos Receive Hashring

Our first exercise will be to deploy a Thanos receive hashring along with the Thanos receive controller.
This simple yet effective stack will help us build a foundation for more advanced configurations later.

1. To start, let's create the `thanos` Namespace:
    ```shell
    kubectl create ns thanos
    ```

1. Create the resources in the directory for the first demo:
    ```shell
    kubectl apply -f 1
    ```

1. Let's take a look at the Pods we created:
    ```shell
    kubectl get pods -n thanos -o wide
    ```

1. Now, let's see the configuration that the Thanos receive controller generated:
    ```shell
    kubectl get configmap -n thanos thanos-receive-controller-tenants-generated -o jsonpath="{.data['hashrings\.json']}" | jq
    ```

1. Our Pods are running and the configuration was generated, so now let's test the 1-node hashring; in another terminal window run a process to make remote-write requests against the Thanos receive service:
    ```shell
    docker run --rm quay.io/observatorium/up --endpoint-type metrics --endpoint-write http://"$(kubectl get service -n thanos thanos-receive -o jsonpath="{.spec.clusterIP}")":19291/api/v1/receive --period=1s
    ```

1. Let's scale up the hashring to three replicas:
    ```shell
    kubectl -n thanos scale statefulset thanos-receive-default --replicas=3
    ```

1. Now let's check the Pods and the generated configuration:
    ```shell
    kubectl get pods -n thanos -o wide -w
    kubectl get configmap -n thanos thanos-receive-controller-tenants-generated -o jsonpath="{.data['hashrings\.json']}" | jq
    ```

## Autoscaling Hashring

In this exercise, we'll attempt to deploy an autoscaling Thanos receive Hashring, where the hashring grows to accommodate increased request load.

*Note*: these instructions assume that the Kubernetes [metrics-server](https://github.com/kubernetes-sigs/metrics-server), or something else implementing the `v1beta1.metrics.k8s.io` resource metrics API, e.g. the Prometheus Adapter, is installed on the cluster.

1. Begin by creating the `thanos` Namespace:
    ```shell
    kubectl create ns thanos
    ```

1. Next, let's deploy the resources from directory for the second demo:
    ```shell
    kubectl apply -f 2
    ```

1. While these are being created, let's take a look at the differences from the first demo.
You should see that the Thanos receive Pod template now includes a sidecar called `configmap-to-disk`.
We use this sidecar to automatically synchronize the hashring configuration file from the API whenever the ConfigMap changes, without having to wait for the eventually consistent Kubelet volume mount to be update.
This allows the Pods in the hashring to react faster to scaling events.
You should also see that we defined a HorizontalPodAutoscaler resource that scales the hashring based on CPU utilization
This resource also clamps the minimum amount of replicas in the StatefulSet to 5, since we want to make sure we always have _some_ ingestion capacity available.

    _Note_: in order to scale based on CPU utilization, we must set resource requests on the containers being scaled.
    ```shell
    $EDITOR 2/thanos-receive-default-statefulset.yaml 2/thanos-receive-default-hpa.yaml
    ```

1. Let's take a look at the Pods we created:
    ```shell
    kubectl get pods -n thanos -o wide
    ```

1. Again, if everything rolled out successfully, we should see that the Thanos receive controller generated successfully generated a hashring configuration:
    ```shell
    kubectl get configmap -n thanos thanos-receive-controller-tenants-generated -o jsonpath="{.data['hashrings\.json']}" | jq
    ```

1. Let's keep a process running to watch how our Pods are rolling out:
    ```shell
    kubectl get pods -n thanos -w
    ```

1. In a new terminal, let's watch the scaling resource for the hashring:
    ```shell
    kubectl get horizontalpodautoscalers.autoscaling -n thanos thanos-receive-default -w
    ```

1. Finally, in another terminal, let's generate some serious load to test our stack.
This time around we'll use Fresh Track's [Avalanche](https://github.com/open-fresh/avalanche) project to generate remote-write requests.
This process will make 1000 remote-write requests where each request has 100k time series.

    *Note*: we are using a fork of the project that doesn't cause the process to exit when it encounters an error.
    This should serve as an indication of how we expect the exercise to go.

    Run the Avalanche container and point it at the Service IP:
    ```shell
    docker run --rm squat/avalanche:log-errors --remote-url http://"$(kubectl get service -n thanos thanos-receive -o jsonpath="{.spec.clusterIP}")":19291/api/v1/receive --remote-requests-count=1000 --metric-count 1000 --series-count 100
     ```
