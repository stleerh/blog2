# Announcing Network Observability

If you are on an airplane and look down on a metropolis on a busy day, you will
see thousands of cars, trucks, and motorcycles driving from every which way at
various speeds.  There will be traffic jams in some places, cars racing across
the highways, other cars completely stopped at intersections, and vehicles
parked at various locations. If you focus on a single vehicle, you can even
track its starting point and its final destination.

With the release of OpenShift 4.12, we are introducing a new feature called
Network Observability.  Like this airplane view, you will be able to visualize
the moving traffic.  Instead of vehicles, it is the data that's moving in your
Kubernetes cluster.  Prior to 4.12, you could only see snapshots of this
traffic.  It is the difference between taking a picture versus filming a video.

Why is this important?  Because with a video, you now have a record of
everything that happened.  This connects all the missing dots to complete the
whole story because you have a timeline of events.  When there is a problem
such as a traffic jam (latency), you will know what happened, who was involved,
when it occurred and for how long, where it happened, and with analysis, you
might be able to figure out why it happened and how to prevent it from
happening again in the future.  Perhaps to avoid this latency that tends
to happen at a certain time of the day, you could increase the amount of
bandwidth only for this period of time to improve utilization and do better
cost analysis and planning.


## Network Observability Architecture

The core of Network Observability is to collect network flows or more
specifically IPFIX data and then present this in powerful visualizations.
NetFlows, as they were originally called, existed back in the 1990s as a way to
capture that network insight at layer 3/4 of the OSI model.

What is unique about our solution thirty years later is that instead of having
a typical router or switch export IPFIX data, an eBPF agent was developed to
hook into the network events so it can capture and export data coming in and
out of the interfaces at the kernel level.  eBPF is a relatively new technology
that allows a program to run in a sandboxed environment, thus extending the
kernel in a safe and secure manner.  Yet, it is not that new as the eBPF agent
will work as far back as Linux kernel 4.18 which was released in 2018.  The
immediate benefit of the eBPF approach is that it is more performant than the
router/switch solution.  In addition, it is not dependent on a particular CNI
(Container Network Interface) such as OpenShift SDN or OVN-Kubernetes.

On the receiving end of this exported data is a flow collector called the
Flowlogs Pipeline (FLP) that processes this data, enriches it to be
Kubernetes-aware, and deduplicates redundant and less relevant data.  If you
have bursty or high amounts of traffic, you will want to consider installing
Apache Kafka to be the middleman in between the eBPF agent and FLP to help with
buffering and improving streaming.

Network Observability also includes a Console Plugin that extends the
browser-based Web Console.  FLP sends the data to Loki to write out to storage.
Loki provides an API for the Console Plugin to query for information to be
displayed.  A high level diagram looks like this:

                                           Web Console &
                                           Console Plugin
                                                /|\
                                                 |
  eBPF Agent -> [Kafka] -> Flowlogs Pipeline -> Loki
               (optional)                        |
                                                \|/
                                            object store


## Installing Network Observability

If you create an OpenShift 4.12 cluster, you will not find Network
Observability because it is an optional operator that needs to be installed
separately.  However, it is included with the self-managed
[Red Hat OpenShift Container Platform](https://www.redhat.com/en/technologies/cloud-computing/openshift/container-platform)
offering at no extra cost.  The good news is that since it is separate from the
core platform, it is supported retroactively back to OpenShift 4.10!

The [Network Observability docs](https://docs.openshift.com/container-platform/4.12/networking/network_observability/network-observability-overview.html)
provide a great step-by-step guide on installing Network Observability.  Now
that you've seen the architectural diagram, you know all of the significant
components that are involved.  The Network Observability Operator only manages
eBPF Agent, Flowlogs Pipeline, and Console Plugin.  The main items to consider
for installation are:

1. Provide an object store
See the [list of supported object stores](https://grafana.com/docs/loki/latest/operations/storage/).
You will need to create a secrets file to access the object store.

2. Install Loki
The recommendation is to install Loki Operator 5.6, which simplifies the
deployment of Loki in microservices mode that is necessary for scalability.  In
Web Console, you can do this from OperatorHub.  Note that if you already have
Loki installed for another purpose, it cannot be shared.  You must still
install a separate Loki for Network Observability.

3. Decide if you need Kafka
For clusters with ten nodes or less, you can try without Kafka.  If you have
25+ nodes, then most likely, you will need it.  Anything in between depends on
the volume of network traffic.  If you install Kafka and accept all or most of
the default values, it will handle about 5K flows per second (fps) and take
less than 5% of your current CPU and memory resources.  Kafka can be installed
using the Red Hat AMQ Streams operator in OperatorHub.

4. Decide on sampling rate
The fps is affected by the sampling rate.  Sampling refers to the ratio of
packets that are evaluated. It defaults to 50, meaning one out of 50 packets
are considered and the rest are ignored.  A value of 0 or 1 means no sampling
and all packets are considered.  When you create a Flow Collector resource from
Network Observability Operator, the sampling field is in the eBPF section.

5. Consider scalability
If you have high network traffic volume, check the documentation on the list of
parameters to change to better support scalability.  We've tested up to 120
nodes with 100 pods each, no sampling, and traffic continuously running on 10%
of the pods.  This consumed between 8-12% additional CPU and 4-8% additional
memory.


## Using Network Observability

Network Observability is only available to users with the cluster-admin role,
such as kubeadmin, since this user can see all the traffic in the cluster, both
infrastructure-related and all applications.  It handles one cluster so any
traffic that goes out or comes into this cluster is considered external traffic
even if the traffic is from another cluster that you own.

Once you have Network Observability installed and have created a FlowCollector
resource, behind the scenes, the network flow data will be created, collected,
, enriched with Kubernetes-related information such as the namespaces and pod
names, and then saved to object store.  The Web Console will pop up a dialog
asking you to refresh the web page.  After that, a new menu item under
**Observe** called **Network Traffic** appears.

![Network Traffic](images/network_traffic_panel.png)
_Figure 1: Overview_

Above the charts, there are common settings that apply to the three tabs near
the top called Overview, Traffic flows, and Topology.  The first dropdown is
Query options.  In it, you can decide what flows are shown.  By default, it is
**Destination** which means it is the ingress traffic to the node as opposed to
**Source** which is the egress traffic.  You typically don't want **Both**
since it will end up reporting the same flow twice, but that may be necessary
if you need to know exactly where the traffic flowed into and out of the
interfaces.  There is also a choice for how you want to match filters (more on
that later) and the maximum number of flows to retrieve.

Next is Quick filters.  They default excludes infrastructure traffic so if
you have a new cluster with no applications running, there will be no data.
The next field provides a powerful filtering capability.  Select a choice
such as Common Namespace and enter a value to build your filter.  If you add
multiple values for the same filter, it will assume this is an OR operation.
Between two different fields, it is an AND operation.  This is where the Query
options can change this also to an OR operation.  In the dropdown, the word "Common" indicates that the field value can be on the source or destination side.

There are links like **Expand** that maximizes the space for this panel.  In the
upper right corner, you can set the time range for the data and have the
panel refresh automatically at various intervals if desired.

The three tabs present different visualizations for the traffic flows.  In the
Overview tab, there are a number of different chart types that gives you a
summary of the bandwidth usage.  The Traffic flows tab presents a detailed
table of each flow enriched with Kubernetes metadata and the ability to choose
what columns to display.  Finally, the Topology tab raises the bar on the user
interface by providing a graphical representation of traffic flows.


## Use Cases

Now that you have a general overview of Network Observability, let's look at
some concrete things you can do with it.  I will go over three possible use
cases, and encourage you to try these out or come up with your own use cases.

### Use case #1: See what's running on my network

Kubernetes is great for orchestration and supporting microservices and
scability, but one area it doesn't help in is observability.  The extra
infrastructure layer and pods coming and going makes it even more difficult to
see what is happening on your network.  That is all about to change.

With Network Observability, you can see all the network traffic if you clear
the default filter so that traffic generated by Kubernetes is also captured.
In the Overview tab, it shows the top projects using the most bandwidth.  Going
forward, more charts will be added for protocol, ports, ingress/egress traffic
and more.  In the Topology tab, filter on "netobserv" and
"netobserv-privileged" in the Common Namespace to see all the other
pods/services that it communicates with.
(TODO)

(TODO)
- Install and use with Grafana


(TODO - may change)
### Use case #2: Measure latency


### Use case #3: Auditing
(TODO)
- Export to Kafka


## Summary

(TODO)
- Provide network observability
- First major foray into eBPF
- Raise the bar on visualizations in Web Console

