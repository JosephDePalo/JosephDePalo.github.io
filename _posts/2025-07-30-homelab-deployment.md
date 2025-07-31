---
title: Deploying my Home Lab
date: 2025-07-30 12:00:00 -0400
categories: [Lab]
tags: [Home Lab]
image: /assets/img/e76d1fa1.png
---

As an aspiring security professional, there aren't many things better than a
home lab to help you learn and grow. Over the past year or two I've ran some
experiments here and there with my Intel NUC 11, but nothing very serious or
regularly maintained. My recent internship, however, has filled my head with
ideas for different things I want to try out on my own.

As enterprise networks are constantly growing and changing, so too will my lab.
This post will go through what will serve as my foundation for future projects.
It will not be a step-by-step guide, but instead an overview of my current home
lab deployment, any issues I ran into setting it up, and future plans.

## Tool Choice

### pfSense

- Already deployed on my Proxmox server.
- Serves as the network gateway for VMs and enforces segmentation.
- Offers NAT, DHCP, and basic firewalling to simulate enterprise perimeter
  defenses.
- Will later support IDS/IPS functionality via Suricata or Snort.

### Wazuh

- Chosen for its full-featured open-source EDR capabilities.
- Handles file integrity monitoring, log analysis, vulnerability detection,
  and threat intelligence integration.
- The Wazuh manager (LWAZ01) aggregates logs from both Linux and Windows agents
  and performs real-time analysis.
- Integrates with both Splunk and Elastic Stack, offering flexibility in future
  expansions.

### Splunk

- Widely used commercial SIEM platform with extensive support for log ingestion
  and correlation,
- In this setup, acts as the central SIEM, receiving enriched and normalized
  logs from the Wazuh manager,
- Helps visualize alerts and patterns from agent telemetry.

## Deployment Summary

The below table describes the machines deployed in my lab.

| Hostname | OS | Components | User |
| --------------- | --------------- | --------------- | --------------- |
| LWAZ01 | Ubuntu 24.04 | Wazuh Stack, Splunk UF | king-frieza |
| LSPL01 | Ubuntu 24.04 | Splunk Enterprise | king-frieza |
| LUBU01 | Ubuntu 24.04 | Wazuh Agent | john-doe |
| LWIN01 | Windows 10 | Wazuh Agent, Sysmon | bob-brown |

The below diagram shows how data will generally flow. Everything on the
`10.0.0.0/24` network is virtualized and must pass through `pfSense` in order
to reach the internet or my home network (though access to my home network is
restricted to my default gateway to reach the internet).

![Home Lab Diagram](/assets/img/c2782f17.png)

## Issues

OS installations went quite smooth, the Wazuh assisted installation method was
simple to follow and quick, but Splunk posed a bit more of a challenge. The
actual installation of it was not a challenge, but getting `LSPL01` to actually
receive and process the logs sent by `LWAZ01` took some trial and error.

After installing the [Splunk Universal Forwarder][splunk-uf-doc] on `LWAZ01`, I
expected `LWAZ01` would begin any alerts from Wazuh straight to Splunk. To my
surprise, I was getting absolutely nothing on the Splunk end of things. And so
began the troubleshooting.

My troubleshooting methodology was as follows:

1. Confirm packets were being sent by Splunk UF by inspecting its logs at
   `/opt/splunkforwarder/var/log/splunk/splunkd.log`. Splunk UF could not
   establish a connection to `LSPL01` on `TCP/9997`.
2. `ss -tulpn | grep 9997` revealed that no processes were listening on
   `TCP/9997`. Upon doing some research, it turns out that receiving on `TCP/9997`
   must be enabled. After enabling it in the UI, still nothing in Splunk.
3. Inspect the logs on `LSPL01` at `/opt/splunk/var/log/splunk/splunkd.log`.
   Grepping for `9997`, I find the below error log:

   ```text
   ERROR TcpInputConfig [41164 TcpListener] - SSL context cannot be created due
   to missing required serverCert parameter from [SSL] stanza. Will not open
   splunk to splunk (SSL) IPv4 port 9997
   ```

4. Research the error. It seems Splunk needs an `inputs.conf` with an `[SSL]`
   stanza. I know where the certificate is, but it is encrypted and I do not
   know where the password is so I regenerate it. The SSL stanza is
   automatically populated with the correct fields.
5. After restarting Splunk, I'm still running across the same error. I'm not
   having any luck after playing around with additional parameters either.
6. Try disabling SSL. This works and logs are now being received on `LSPL01`,
   but forwarding is no longer encrypted.

For now I've left SSL disabled. This, however, is an issue I intend on
returning to when I dive into Splunk hardening. Ideally, I would like to have
mutual authentication between Splunk server and forwarders among other things.

## Conclusion

With the foundation of my lab built out, I can now move onto making
improvements and building out additional functionality when time allows. Below
I've listed a few of the things I intend on working on, though I'm quite
limited on RAM right now (With all of the above virtual machines running, my
server is running at 29/32 GB):

- Endpoint Hardening
  - Wazuh has a compliance checking module that allows me to compare endpoint
    configurations to their respective CIS hardening benchmarks.
- Active Directory
  - To move beyond local user accounts and more adequately mimic an enterprise
    environment, I can deploy a domain controller on my network and join all of
    my machines to it.
- Splunk Hardening
  - As mentioned earlier, there is still more I want out of Splunk. I'd like to
    take a look at what else I can do with it since I've barely even scratched
    the surface.
- Create Alerts & Test with Atomic Red Team
  - What good is a SIEM and EDR if you're not using and customizing them? I'd
    like to perform some malicious activities on my network to see what gets
    picked up, what doesn't, and how I can set up useful, accurate alerts.

There is a lot I want to do in this lab environment, so this post likely won't
be my last. Stay tuned for more.

[splunk-uf-doc]: https://docs.splunk.com/Documentation/Forwarder/latest/Forwarder/Abouttheuniversalforwarder
