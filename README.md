***Kubernetes Logging using EFK***

Logging for Kubernetes Clusters

This time let's create a 3 Node Cluster using the kind-config.yaml file. 

    kind create cluster --name tambootcamp --config=kind-config.yaml

Wait till the cluster is up and running 

Let's install the metrics-server using helm. 

    helm install metrics bitnami/metrics-server


Since I installed the metrics server, I thought I can have a look at the logs of the metrics-server deploy using 

    kubectl logs deployment/metrics-metrics-server

I noticed that I was getting errors there like: 

unable to fully collect metrics: unable to fully scrape metrics from source kubelet_summary:kube: unable to fetch metrics from Kubelet kube (kube): Get https://kube:10250/stats/summary/: x509: certificate signed by unknown authority

A bit of google search and it led me to the solution of editing the deployment to add --kubelet-insecure-tls

I did an edit of the deployment using 

    kubectl edit deployment metrics-metrics-server

and added 

    - --kubelet-insecure-tls 

under - args within containers under spec

Now did a helm upgrade on it using: 

    helm upgrade --namespace default metrics bitnami/metrics-server --set apiService.create=true

Within a few minutes, I was able to get it working. You can check this by running 

    kubectl top nodes

This also works for pods. 

Now you can run the command which was shown after the helm command and see it's output: 

    kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes"

Ok. Let's fire up octant now. 

Start Octant by typing(on another terminal tab):

    octant

You would notice that octant now starts showing utilization details on the pods! 

Alright let's get started with the EFK Stack now. 

Let's deploy ElasticSearch first, we will create a namespace called as logging using

    kubectl create ns logging

then deploy elasticsearch using 

    kubectl apply -f elasticsearch.yaml

Wait for the rollout to complete. This is a statefulset with 3 replicas so it is going to deploy 3 pod es-cluster-0 thru 2 so this would be the right time to refill that coffee mug! 

Check the status by running

    kubectl get all -n logging 

Once all the 3 pods are deployed, a good way to check this would be to port-forward the elasticsearch service through octant and having a look at the link created. 

To verify if we are getting log information in there run 

    curl http://localhost:9200/_cluster/state?pretty 

You would see lot of information in there -> Which is a good sign! 

Ok, let's proceed by deploying a fluentd which is a log collector and helps with the better understanding of them. 

    kubectl apply -f fluentd.yaml

Now let's deploy Kibana which is a cool data visualization tool 

    kubectl apply -f kibana.yaml

Give it a few minutes and then make sure that everything in the logging namespace is showing as "Running" 

Let's deploy a sample application which simply echo's date and time on the stdout. 

    kubectl apply -f counter.yaml

Once up and running. You can port-forward the kibana service on octant and access the Kibana UI. 

Once the Kibana UI is accessible, click on **Explore on my Own** and then the **Discover** icon in the top left, in the Index pattern text box, type “logstash-*” and click **Next step**. 

In the configure settings, choose “@timestamp” from the drop-down and click **Create index pattern**. 

Now click Discover again, we should now see live log entries from our cluster and pods. 

Play around, it's not exactly vRealize Log Insight, but kinda same look and feel ;) 

Try searching for the counter pod logs! 

Let's take a look at installing Prometheus and Grafana now. 

You can easily install Prometheus and Grafana using helm. 

    helm install myprom bitnami/kube-prometheus

    helm install mygrafana bitnami/grafana

Give it a few minutes, check out its rollout on octant. 

Get the admin password for grafana using 

    kubectl get secret mygrafana-admin --namespace default -o jsonpath="{.data.GF_SECURITY_ADMIN_PASSWORD}" | base64 --decode

Dont include the "%" when you copy the password. Save it on your notepad, we will first have a look at Prometheus. 

Have a look aaround in there, start with the Status tab. Look at the targets and others. 

Now let's try a simple PromQL query there on the graphs. Type the below in the search box to see the Return all time series with the metric http_requests_total.


    kubelet_http_requests_total

Play around with it , try some queries from https://prometheus.io/docs/prometheus/latest/querying/examples/ 

Now onto grafana, port-forward the mygrafana service in the default namespace in octant. 

The username is admin and the password is the password you copied on to your notepad(without the %)

First let's add Prometheus as a datasource to Grafana. 

Go back to your terminal and list all the services there using 

    kubectl get svc -A

Look for the service "myprom-kube-prometheus-prometheus" and copy the ClusterIP so that the URL would be http://**<CLUSTER_IP of myprom-kube-prometheus-prometheus>**:9090

Leave everything as is and click Save and Test. It should say Datasource is working. 

Now let's have a look around importing some readymade dashboards , one of my favourites ones is 11802. 

Click on **Dashboards** and then **Manage** and then **Import** and enter 11802 and click **Load**. 

Now go back to **Manage** and then select 1-kubernetes-cluster-overview and have a look :) 

That's it for today's logging session. See you next time. 
