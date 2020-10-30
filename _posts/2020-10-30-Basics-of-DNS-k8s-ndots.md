---
layout: post
title: Basics of DNS lookups in Kubernetes - ndots
---

One of the fundamental technologies which are used in Kubernetes is DNS. Most of the components and concepts in this ecosystem rely on lookups provided by it.  
In this post I'll try to explain a little bit a whole process and point out some common configuration issues.

Worth noting is fact that Kubernetes have built-in service discovery mechanism. By default pods can communicate with each other using convention form   
`service-name.namespace.svc.cluster-domain.example` or `pod-name.namespace.pod.cluster-domain.example` (Services can be headless or not headless).  
This convention is just DNS A record assigned to resource at the time of creating. 

## DNS components for Kubernetes
Nowadays the most popular DNS Server is CoreDNS. Mainly due to excellent integration with major cloud providers and countless amount of plugins which  
extends standard behaviour of that project.

To make the next points understandable, I will describe briefly what is going on in runtime process of CoreDNS.
* Pods and service of CoreDNS are deployed
* CoreDNS loads plugins specified in configuration. If no configuration was given then it only loads `whoami` and `log` plugins
* Starts listening on port 53 unless it's overrided using `-dns.port` argument
* If a Pod's `dnsPolicy` is set to `default`, it inherits `resolv.conf` configuration from node.
```
nameserver 10.20.0.20
search namespace.svc.cluster.local svc.cluster.local cluster.local 
options ndots:5
```

The configuration above is really interesting, pretty simple and at the same time it can cause a lot of trouble. Let's break it down into parts.  
`nameserver` - Cluster IP of CoreDNS service.  
`search` - List of completions for inter cluster communication.  
`ndots` - Minimal value of dots in query to qualify it as FQDN.

## What exactly does it mean that query is FQDN. What is ndot?
FQDN (Fully qualified domain name) is a query which informs Kubernetes to not perform local DNS lookups. Basically the query is treated as absolute  
(replacment name) when it contains `.` at the end.  
Otherwise Kubernetes will perform local lookups on the end of which it will stick the domains from `search` section. Such lookups will continue as long as
* One of the local lookups result with `NOERROR`
* It finishes iterating over search paths
If each of the iterations result with `NXDOMAIN` then Kubernetes will perform inital query as FQDN.

Fortunately Kubernetes provided one more solution to omit local searches. As I mentioned earlier, `ndots` is an minimal value of dots in DNS query to be  
recognized by Kubernetes as FQDN. By default `ndots` value is set on `5` and if you think about it seems as pretty high value. Indeed, it's high but it's  
done in that way to ensure that all of the local lookups will reach desired target. If you take the look into form convention of services and pods you will 
notice that it contains up to 4 dots.  

## Reduce NXDOMAIN errors
Over time, clusters grow, so at the same time number of queries to DNS grow. If most of your traffic is external it means that probably 3/4 or 4/5 (depends on number of entries in search section of resolv.conf)  
will result with `NXDOMAIN` due to fact that most of the domains contains only one dot (so local lookups will be performed). In situation like this it can be fixed on two different ways.

1. Append at the end of each external request `.`
2. Change the `ndots` value for your pod using `dnsConfig`

```
dnsConfig:
  options:
    - name: ndots
      value: "2"
```

## Further Readings
* [Overview of Kubernetes DNS support](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
* [Specification for DNS-based Kubernetes service discovery](https://github.com/kubernetes/dns/blob/master/docs/specification.md)
* [Kubernetes service discovery in examples](https://platform9.com/blog/kubernetes-service-discovery-principles-in-practice/)
