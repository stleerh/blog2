# Announcing Network Observability

If you've been on an airplane and look down on a metropolis on a busy day, you
will see thousands of cars, trucks, and motorcycles driving from every which
way at various speeds.  There will be traffic jams in some places, cars
completely stopped at intersections, light traffic on the outskirts of the
city, and a bunch of vehicles parked at corporate buildings, stores, and
stadiums.  If you focus on a single vehicle, you might be able to track its
starting point and how it reached its destination.

With the release of OpenShift 4.12, we are introducing a new feature called
Network Observability.  Like this airplane view, it will give you this grand
picture of moving traffic, but instead of vehicles, it will be data
moving in your Kubernetes cluster.  Why is this important?  Because prior to
4.12, you could only read about what happened in disparate logs and events.
There is some monitoring at the lower level, but you could not visualize and
connect the dots to see why something happened because that piece of data was
missing.  If you access a web site and it takes much longer than expected to
complete a transaction, you have very little insight into where that latency is
coming from.  If you want to see which app or pod is talking to other apps,
pods or Internet, you simply did not have that information.  If you don't
understand why you are paying so much for your infrastructure, you can now
delve deeper and do better cost analysis and planning.  This is just the tip of
the iceberg.


## Network Observability Architecture

The core of Network Observability is to collect network flows or more
specifically IPFIX data and then present this in powerful visualizations.
NetFlows, as they were originally called, existed back in the 1990s as a way to
capture that network insight at layer 3/4 of the OSI model.

What is unique about this solution is that instead of having a typical router
or switch export IPFIX data, an eBPF agent was developed to hook into network
events so it can capture and export data coming in and out of the interfaces at
the kernel level.  eBPF is a relatively new technology that allows a program to
run in a sandboxed environment, thus extending the kernel in a safe and secure
manner.  The immediate benefit of this approach is that it is no longer
dependent on a particular CNI such as OpenShift SDN or OVN-Kubernetes, and it
is more performant than the router/switch solution.

On the receiving end of this exported data is the Flow Collector that processes
this data, enriches this data to be Kubernetes-aware, and deduplicates
redundant and less relevant data.  If you have bursty or high amounts of
traffic, you will want to consider installing Apache Kafka to be the middleman
in between the eBPF agent and Flow Collector to help with buffering and
improving streaming.

The Flow Collector sends the data to Loki to write out to storage.  Loki also
provides an API for Web Console to query for information to be displayed to the
user.  A high level diagram looks like this:

                                          Web Console
                                             /|\
                                              |
  eBPF Agent -> [Kafka] -> Flow Collector -> Loki
               (optional)                     |
                                             \|/
                                          object store



## Installing Network Observability

If you install OpenShift 4.12, you will not find Network Observability because
it is an optional operator that needs to be installed separately.  However, it
is included with the self-managed [Red Hat OpenShift Container Platform](https://www.redhat.com/en/technologies/cloud-computing/openshift/container-platform) offering
at no extra cost.  The good news is that since it is separate from the core
platform, it can be installed retroactively back to OpenShift 4.10!

The [Network Observability docs](https://docs.openshift.com/container-platform/4.12/networking/network_observability/network-observability-overview.html) provide a great step-by-step guide on installing Network Observability.  Now that you've seen the architectural diagram, you know all of the significant components that are involved.  The Network Observability Operator only manages eBPF Agent, Flow Collector and Web Console.  The main items to consider for installation are:

1. Provide an object store
See the [list of supported object stores](https://grafana.com/docs/loki/latest/operations/storage/).  You will need to
create a secrets file to access the object store.

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

Once you have Network Observability installed, behind the scenes, the network
flow data will continually get collected, enriched with Kubernetes-related
information such as the namespaces and pod names, and then saved to object
store.  The Web Console will pop up a dialog asking you to refresh the web
page.  After that, a new menu item under **Observe** called **Network Traffic**
appears.

At the top of the Network Traffic page, there are common settings that apply to
the three tabs just below it called Overview, Traffic flows, and Topology.  The
first setting is a dropdown for Query options.  One such option is to consider
ingress traffic, egress traffic (default), or both.  You typically don't want
both unless you have a need to see exactly how the traffic is flowing in and
out of an interface.

Next is Quick filters which, by default, excludes infrastructure traffic so if
you have a new cluster with no applications running, there will be no data.
The field next to it provides a powerful filtering capability.  Select a choice
such as Common Namespace and enter a value to build your filter.  If you add
multiple values for the same filter, it will assume this is an OR operation.
Between two different fields, it is an AND operation, although this can be
changed in Query options to be OR.  The "Common" wording implies that the field
value can be on the source or destination side.

Click the Expand link to maximize the space for this panel.  In the upper right
corner, you can also set the time range for the data and have the panel refresh
automatically at some interval if desired.

The three tabs present the traffic flows using different visualizations.  In
the Overview tab, there are a number of different chart types that gives you a
summary of your network traffic.  The Traffic flows tab presents a detailed
table of each flow enriched with Kubernetes-related information and the ability
to choose what columns to display.  Finally, the Topology tab raises the bar on
the user interface by providing a graphical representation of traffic flows.


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

### Use case #2: Measure latency



### Use case #3: Auditing



Future: Automation, integration


Do note that in some case when it reaches capacity, you will have downtime on the Network Observability project



## Summary

We have yet to realize all the benefits and advantages that we can gain by
using eBPF technology.
