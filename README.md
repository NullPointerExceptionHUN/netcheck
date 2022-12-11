# netcheck

This aims to be come a simple but powerful script to assess the quality of
an Internet uplink. Once functional, you'll be able to use it as a building
block for an uplink failover solution.

It sends a mix of packets out via an interface and measures what percentage
receives a reply.

## Connectivity tests performed

Eventually, the following test types will be supported:

 * ICMP Echo
 * TCP SYN
 * DNS over UDP
 * NTP over UDP

## Background and Motivation

It is becoming more and more common to have more than one uplink, provided
by more than one ISP, but no provider independent IP range and no AS; so
while your connection is redundant, there is no obvious way of making sure
you route out packets over a link that is alive.

One approach would be to send out connection requests (TCP SYN packets, DNS
queries etc.) over several uplinks simultaneously, then wait for the first
reply and drop the other connections. This would need kernelspace support.

Another approach is to monitor your uplinks to see which ones actually work,
and only route packets out over the ones that are "good enough".

The Linux policy routing framework makes it easy to send packets out via a
specific route: you can mark packets using `SO_MARK`, and then have `ip
rules` to route specially-marked packets out via specific interfaces. `SNAT`
netfilter rules can be used to make sure the source address is appropriate
for the chosen uplink.

What has been missing so far was a tool that would be able to generate test
traffic.

One of the authors of this `netcheck` script experimented with wrapping
`oping -m` in a shell script, but some ISPs enforce rate limits on ICMP,
which made the results unrealiable. He also tried using an `LD_PRELOAD`-able
library to `SO_MARK` the packets `nmap` generates, and then use `nmap` to
connect to a bunch of normally highly available webservers, but the tests
were too bursty and the results were once again somewhat unreliable.

Thus, the idea for this script was born.

## Specification of functionality

 * Embrace The Unix Way: do one thing and do it well, but be interoperable with other tools. Thus, we don't want this script to be a complete link failover solution, just the component that decides whether a particular Internet link "works".
 * Send a continuous, low-rate stream of test packets, all marked using SO_MARK (if that is availble), and continually track the amount of replies we receive.
   * We explicitly don't want to wait to see if we receive a reply before we send out the next packet, so this probably means using `asyncio`.
 * Make the ratio of successful tests over the last `n` seconds or `n` packets available to external programs using some mechanism (e.g. readable from a FIFO, our stdout, or perhaps even a Unix domain scoket). This is our interface to the rest of the failover logic.
   * Alternatively or additionally, report the time the last reply was received, or the number of test packets send that haven't yet been replied to.
   * When considering which packets have received replies, we should consider a timeout, which can be different depending on test type (e.g. higher for DNS); don't count tests whose timeout hasn't expired yet towards the ratio of unsuccessful tests.
   * Optionally, we may support calling some hook under certain conditions; e.g. write a "1" or a "0" into a file depending on whether we think our uplink is "up" or "down".
 * Be able to run well under a service supervisor like `runit`. If this also allows us to cooperate well with `systemd`, that's a bonus but not a priority.
 * Log messages to stderr, with log verbosity being configurable and preferably changeable at runtime.
   * Log messages should be structured, meaningful and rich; e.g. if we can't reach a webserver, we want to be able to log not only its IP, but also what website we expected it to serve.
 * The mix of test traffic should be configurable in some simple way, e.g. by assigning relative weights to the different tests.
 * Where we send our test traffic should be configurable at runtime.
   * The script should shun test destinations that proved unreachable, for some configurable time (or until it runs out of destinations for a particular kind of test, whichever happens sooner).
 * It is not necessary for a single instance of the script to be able to monitor multiple uplinks; the user can just run one instance per uplink.
 * Determine whether TCP SYN tests should consider both an RST and a SYN ACK a success, as well as possibly ICMP messages generated by "something near the destination" (since this would still mean that the link works, only the test destination doesn't).
 * Support both iterative and recursive DNS queries in tests. Idea for configuration and implementation:
   * It doesn't matter what the reply says as long as there is a reply (we're only testing connectivity).
   * For iterative queries, the configuration can specify a set of {DNS server IP;domain} tuples to generate queries for the given domain and send them to the given server. (The queries can always reference the same domain; there is no need to randomize them.)
   * For recursive queries, we need a list of normally-resolvable domain names and a separate list of DNS server IPs. We pick a random entry from both lists to generate a query.
   * In all cases we can always query the SOA record, because:
     1. if it exists, it won't be very large (`A` can have an arbitrarily large result set and we don't want to generate unnecessary traffic);
     2. even if it doesn't exist, querying it shouldn't trigger and IDS/IPS (while looking for e.g. `HINFO` records might).
 * Idea for configuration/implementation of TCP SYN tests:
   * Use the filesystem as a database:
     * Have one directory per TCP port.
     * Have files named for IP addresses in each directory. If tcp/80/1.2.3.4 exists, we expect to be able to connect() to 1.2.3.4:80.
     * Have the files contain some text that can be used in logs; e.g. the name of a website hosted on that IP.

## Current state

The script currently is a rough proof of concept, and doesn't use `asyncio`
yet.

