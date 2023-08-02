# 🧅 ESXi-Security-Onion-Passive Collection Platform 🧅

## Introduction

This showcases a project similar to what I have built and maintained in my professional work as a Junior Security Engineer while in the military. This cluster is meant to be used on a network as an additional security analysis tool for cyber threat hunting. 

This platform is based off ESXi hypervisors using several virtual machines of a Distributed Build of Security Onion version 2.3.100. 

A Security Onion Distributed Deployment contains the following:
- 1 Master/Manager Node
  - For a central management server in the distributed setup. Responsible for controlling activities of all other nodes in the deployment. 
- 1 Sensor/Forward Node (minimum)
  -  For capturing and processing network traffic and security events with Zeek/Suricata. 
- 1 Search Node (minimum)
  -  For storing and indexing the network logs generated by the Sensor Nodes.

Here is a picture showing the minimum build of a Security Onion Distributed Deployment from the [Security Onion Documentation Website](https://docs.securityonion.net/en/latest/architecture.html):

![image](https://github.com/gervguerrero/ESXi-Security-Onion-Passive-Collection-Platform-/assets/140366635/5cbc6295-e12d-4522-8653-ef46a7b6b2bd)

The highlight of a Distributed Builds allows for greater scalability and performance when adapting to different customer network sizes. Larger networks require more nodes to process a larger volume information, while smaller networks require less. Scaling up and down for Sensor/Forward and Search Nodes is as easy as plug and play to introduce your new asset to the Security Onion cluster, as long as they are built with the same Security Onion version. 

## Customized ESXi Security Onion Build 

![data collection platform purple](https://github.com/gervguerrero/ESXi-Security-Onion-Passive-Collection-Platform-/assets/140366635/fa6ad362-b50f-45a1-acc7-b10ea8f31acf)

Shown above, we can see an overview of the Security Onion cluster collecting raw network traffic from a client's network via SPAN/Mirror ports. 

There are 3 Client Network switches each with a SPAN/Mirror port to show different areas of the network. Each Client switch feeds directly to an ingest monitoring port for a Sensor Node (Light Server). The data is then forwarded to the Manager and Search nodes for an enrichment process running through the Elasticsearch Stack (ELK).  

The Heavy server with ESXi houses a Manager Node and 3 Search nodes where the data after some parsing ultimately sits. The end user Security Analyst on their workstation runs queries in the Kibana GUI web page to pull data out of those processed Elasticsearch indexes, to return data in a visual format via tables, graphs, or other tools  available through the Kibana GUI. 

To see how network defenders would use Kibana in Security Onion to identify network anomalies or threats, see my page: [LLMNR-NBT-NS-Poisoning-DETECTION](https://github.com/gervguerrero/LLMNR-NBT-NS-Poisoning-DETECTION).

## Data Flow

In a simple explanation, the Sensor Nodes analyze the network traffic coming off the Switch SPAN/Mirror port with Zeek, Suricata, and Strelka, then sends it through the Elasticsearch stack ELK via Filebeat. It first passes through Logstash and Elasticsearch instances on the Manager Node, and then the Search Node's Logstash and finally stored in the Search Node's Elasticsearch

![data collection platform purple 2](https://github.com/gervguerrero/ESXi-Security-Onion-Passive-Collection-Platform/assets/140366635/2da5de2f-8153-4d3d-a3f2-0f40132eb52f)

**For a more detailed explanation:**

Once the traffic from the client network (from either SPAN/Mirror Port or In-Line TAP) is ingested into the Sensor Node first, a Berkeley Packet Filter (BPF) can be applied that works with Suricata on the NIC level to filter unwanted traffic collection.

Working with the NIC, the data stream reaches **AF_Packet**. AF_Packet is a Linux Kernel tool that allows direct packet capture from the NIC bypassing standard socket mechanisms. Security Onion uses it for high speed and low latency packet capture and creates 4 separate data streams for each service being used:

- **Zeek**: A metadata protocol analyzer and categorizer  
- **Suricata**: IDS/IPS tool with protocol/metadata analyzing features 
- **Strelka**: File analysis framework for detecting malware using Yara rules 
- **Steno**: Captures and stores the PCAP, working with Sensoroni to deliver the PCAP for queries in the S.O. Console

**Sensoroni** is just the background service that structures the Security Onion (S.O.) Console web page, and also returns some data on the status of each Security Onion node to display in Grid and Grafana. It also works between the Sensor Node and Manager Node to return a specified timeline of PCAP that a Network Analyst queries in the S.O. Console. 

Once each of the 4 services does their job with the data stream and their specific logs are created, **Filebeat** (a log shipper) sends those logs to the Manager Node to be transformed and mutated by the Manager Node's **Logstash**. 

It will then pass through the Manager Node's Elasticsearch, and then gets redistributed by **Redis** to the Search Node's Logstash. It will then ultimately be indexed and stored by Elasticsearch in the Search Node as a database. 

Now that the has been parsed, enriched, and stored in Elasticsearch in the Search Node, a Security Analyst using **Kibana** on their workstation will query that data. The Search Nodes will search their Elasticsearch database and answer the query delivering the information to Kibana. Kibana will display the information with visuals defined by the analyst, allowing for flexible streamlined threat hunting.

The **Curator** service on the search node is just to refine and manage elasticsearch indexes.

Our services in each node is color coded based off experience with managing services in the past.

- Blue = Services that may be altered by Security Engineers
- Green = Services/Tools that are expected to be used or managed by Analysts and Security Engineers
- Red = Services that should NOT be touched or carefully altered if needed.

An example of a sensitive service in Security Onion is **Salt**.

With Salt, engineers are expected to know how to manage and distribute changes to each SO node through Salt. There are many paths and directories with Salt, it can be tricky to manage properly. A change made to the Manager Node's Salt instance will relay those changes to any dependent Security Onion Nodes in the Distributed Build, which can break a Distributed Build cluster if not managed properly.  


## ESXi Security Onion Build Specs and Usage Case
![data collection platform purple](https://github.com/gervguerrero/ESXi-Security-Onion-Passive-Collection-Platform-/assets/140366635/fa6ad362-b50f-45a1-acc7-b10ea8f31acf)

### Inventory 

**Heavy Server - VM Hosting Server. Resource intensive for Manager and Search Nodes.** 
- Dual Xeon 6138 CPUs, 20 Cores Each (80 Total threads)
- 512 GB RAM
- 7.68 TB SSDs x 12 (92 TB Total, RAID 6 Setup)
- Intel Quad SFP port Optical 10GbE
- Intel X550T2, 2 RJ45 Ports 10GbE
- 2 RJ45 Ports 1 GbE
- 2.0 USB Ports x 4
- VGA Connector

**Light Server - Raw data collection/analyzer, stores PCAP.**
- Xeon-D1587 CPU, 16 Cores (32 Threads)
- 128 GB RAM
- 7.68 x 4 TB SSDs (30.72 TB Total, RAID 5 Setup)
- 2 SFP Ports 10 GbE
- 6 RJ45 Ports 1 GbE
- 2 2.0 & 3.0 USB ports

**Standard Managed Switch.**

**Optional Based On Sensor Strategy/Data Collection Plan: Gigamon GTAP A Series GTP ASF01**
**Provides completely passive full duplex monitoring and absolute fault tolerance.** 
- 4 x 1 GbE Ports
- 2 TAP Points
- Fail Open, Power over Ethernet

### Real World Application

The Security Onion Distributed Build shown above is a similar model that was used to monitor a small to medium business network of around 400 - 500 devices. 

Note that traffic ingestion and storage usage will vary from network to network and depends on where the data collection points are at. 

Security Engineers should take extra caution to understand or avoid data rollover by recording disk usage at a daily rate, and should have backup Nodes or a plan to ship off the data to a storage site. 

(In 2021) One of the issues encountered was discovering disk usage space on a Sensor Node was DECREASING instead of increasing. An investigation revealed that PCAP captured from the start of the cluster deployment was being overwritten. 

Salt's configuration on the Manager Node handling the default PCAP limited a hard cap of 30,000 files. The solution was to change the configuration to a higher number that fit the duration of the limited capture time. 

The file path to this configuration is on the Manager Node: "/opt/so/saltstack/default/salt/pcap/files/config".

Upon altering the config for PCAP, the change was replicated in the /opt/so/conf/steno/config on the Sensor Nodes  and immediately took effect after a restart of steno on the local nodes with “so-restart steno”.

**Resources:**
- [Official Security Onion 2.3 Documentation](https://docs.securityonion.net/en/2.3/#) 
