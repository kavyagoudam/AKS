## Monitoring options for AKS
1. All the stack inside the cluster
    Grafana and prometheus servers and components installed inside the cluster and managed by users
2. All the Stack managed by Azure
    Azure container insights is a managed service that collects  metrics and logs from the cluster
3. All the stack managed by cloud
    Monitoring agents, multiple clusters sends metrics and logs to service provider
4. Azure monitoring with native /oss tools
    Azure container insights integration with prometheus metrics format and grafana dashboards

1. All the stack inside the cluster
    Monitoring AKS using in-cluster deployment
    Great for:-
    Full control over the configuration

    Bad for:-
    Managing  prometheus cluster(disk, backup, updates...)
    Consuming cluster resources
    No control server

    could be optimized by using kubernetes operators - prometheus operator, thanos operator

2. All the Stack managed by Azure
    Monitoring agents sends metrics and logs to Azure monitor
    Great For :-
        Data and servers outside the clusters
        Azure managed service
        One click deployment and support
        Data aggregation for multiple clusters
    Bad for :-
        Less config options for customization
        additional cost for log analytics
        No rich dashboards and alerts

3. All the stack managed by cloud
    All metrics and logs from different apps stored in one central location
    Great For:-
        Easy to install and configure
        Only the agents installed in the cluster
        Support from provider
        Rich set of dashboards and alerts
        Monitoring logging tracebility
    Bad For:-
        Additional cost
        vendor lock-in
    Examples:-
        native integration with azure: datadog, logz.io
        elastic, dynatrace
        splunk, grafana cloud

4. Azure monitoring with native /oss tools
    Container insights integration with prometheus metrics andgrafana
    Great For:-
        Easy to install, configure backup scalibility, data rentention
        Only the agent installed in the cluster
        Native Azure support
        Rich set of dashboards and alerts
        Benefits from the OSS ecosystem
    Bad for :-
        Additional cost
        Vendor lock-in