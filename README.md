
in this repo we will send pfsense log in wazuh ( an open source SIEM ) , so we will dive in pfsense configuration and wazuh configuration and at the end wazuh will decode the log from pfsense and according to a custom rule it will trigger an alert that we will see in the dashboard  .. let's goooooooo



### **Sending pfSense syslogs to Wazuh SIEM**

![Sending pfSense syslogs to Wazuh SIEM](https://marceltc.com/content/images/size/w1200/2024/10/log-data-collection1.webp)

I’ve been setting up a Wazuh SIEM for my homelab, and wanted to get my pfSense routers and firewalls integrated with Wazuh. On most endpoints, you can simply install the Wazuh Agent. On pfSense, however, the agent is not available as an official pfSense package and enabling the FreeBSD repository to install it is neither recommended nor supported. Installing the Wazuh Agent via the FreeBSD repositories may require upgrading pfSense’s pkg version to an unsupported version and may break updates.

Luckily, Wazuh allows importing and processing of syslogs, which will provide both logging and alerting without having to install the agent! This agentless setup has the added benefit of being able to collect and forward syslog data not only from pfSense itself, but other networked devices.

## **The problem: pfSense syslog format**

Simply enabling remote logging to the Wazuh server doesn’t work in pfSense because pfSense does not send hostnames in the syslog headers, breaking Wazuh’s pre-decoder. This is a known bug feature, as RFC 3164 only recommends that hostnames be included, but is not required.

Syslog-ng, however, uses strict RFC 3164 formatting, so implementing syslog forwarding to Wazuh is still possible. I configured pfSense to send its syslogs to Syslog-ng, which will then send correctly formatted logs to the Wazuh server. Here’s how I did it!

## **The solution**

### **Prerequisites**

- A working Wazuh installation
- Wazuh is [configured to receive syslogs](https://documentation.wazuh.com/current/user-manual/capabilities/log-data-collection/syslog.html?ref=marceltc.com) on port 514/udp
- The Wazuh machine’s host firewall allows port 514/udp

First, install Syslog-ng from the package manager. Once installed, enable it and allow it to listen on loopback and any other LAN interfaces you want to collect and forward syslogs for. Everything else can be set to defaults.

![](https://marceltc.com/content/images/2024/10/Screenshot_20240823_082516.png)

Ensure that “loopback” is selected under “Interface Selection"

Next, go to Status > System Logs > Settings. Ensure that the log format is BSD, and enable “Remote Logging” at the bottom of the screen. Under “Remote Log Servers,” put the loopback IPv4 address and the default Syslog-ng listening port (5140). Finally, check “Everything” to send syslog messages for all services.

![](https://marceltc.com/content/images/2024/10/Screenshot_20240823_083038.png)

Remote Logging settings

Let’s ensure Syslog-ng is getting the logs. Back in Syslog-ng, go into the “Log viewer” tab. You should see log messages populated. If not, wait a while (or perform actions to generate some logs) and refresh. Notice that the stored logs include an IP/hostname in the header!

Next, go to the “Advanced” tab. Here, we will set up the actual log forwarding. First, start by creating a destination. This will tell Syslog-ng where and how to connect to the Wazuh server.

![](https://marceltc.com/content/images/2024/10/syslog-ng_destinationconfig.png)

Set your destination and source IP, as well as the transport layer protocol (UDP)

- Set `{ network("WAZUH_SERVER_IP" transport(udp) localip(PFSENSE_LOCAL_IP)); };`
- Ensure that the pfSense local IP matches the `allowed-ips` section in your Wazuh server configuration
- Object name is arbitrary, but you will need it for the next step

Next, create a log object. This will tell Syslog-ng to route the logs from the default logging location (the Syslog-ng log file) to the Wazuh destination that we just set up.

![](https://marceltc.com/content/images/2024/10/syslog-ng_logconfig.png)

Route logs from “_DEFAULT” to “wazuh”

- Set `{ source(_DEFAULT); destination(wazuh); };`
- Ensure that the destination field contains the same object name as what you set above (ex. “wazuh”)

That’s all! Wazuh is now receiving syslog messages from pfSense. Syslog-ng can now also be used to collect logs from other networked devices (like other pfSense firewalls on the network) and forward those to the Wazuh server.

## **Testing logging and alerting rules**

### **Logging**

To ensure that the logs are being received and processed by the Wazuh server, enable the `logall` options in your `ossec.conf` file (see [Wazuh documentation](https://documentation.wazuh.com/current/user-manual/manager/event-logging.html?ref=marceltc.com#archiving-event-logs)). This is for testing and debugging only, so be sure to revert these settings once you’re done (unless you have a lot of disk space)!

Tail the archives file to see the latest syslog messages. Look for a message from the pfSense’s `filterlog` process.

```
$ sudo tail /var/ossec/logs/archives/archives.log
...
Aug 22 18:05:32 pfsense filterlog[73973]: 71,,,12004,igb0,match,block,in,4,0xc0,,1,29888,0,DF,2,igmp,36,192.168.1.254,224.0.0.1,datalength=12
...
```

Copy and paste the message into Server Management > Ruleset Test on the Wazuh dashboard or use the `wazuh-logtest` CLI utility. You should see that Phase 1, 2, and 3 correctly decoded. Wazuh will use the built-in pfSense decoder (also available on their GitHub) to parse the firewall logs.

If you don’t see anything in the `archives.log`, run a packet capture on the Wazuh server host to ensure that you’re actually receiving syslog messages:

```
$ sudo tcpdump -i any port 514 -n
tcpdump: data link type LINUX_SLL2
dropped privs to tcpdump
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
12:00:00.020708 ens18 In  IP 10.8.8.1.5525 > 10.8.8.20.syslog: SYSLOG cron.info, length: 141
12:00:00.030970 ens18 In  IP 10.8.8.1.5525 > 10.8.8.20.syslog: SYSLOG cron.info, length: 86
12:00:00.048290 ens18 In  IP 10.8.8.1.5525 > 10.8.8.20.syslog: SYSLOG cron.info, length: 105
12:00:00.048291 ens18 In  IP 10.8.8.1.5525 > 10.8.8.20.syslog: SYSLOG cron.info, length: 133
12:00:00.048291 ens18 In  IP 10.8.8.1.5525 > 10.8.8.20.syslog: SYSLOG cron.info, length: 85
12:00:00.495196 ens18 In  IP 10.8.8.1.5525 > 10.8.8.20.syslog: SYSLOG local0.info, length: 152
12:00:00.495726 ens18 In  IP 10.8.8.1.5525 > 10.8.8.20.syslog: SYSLOG local0.info, length: 152
```

### **Alerting**

Wazuh comes preloaded with some pfSense alert rulesets. One of the rules checks for repeated firewall blocks from the same IP, and logs an alert with a severity level of 10.

To do this, I used an aggressive `nmap` scan on a Kali Linux VM to scan a blocked subnet and trigger some firewall events.

```
$ nmap -A 10.8.8.0/24 --min-rate 100
```

On the Wazuh dashboard, navigate to the Medium Severity alerts, and you should see a new alert from the pfSense `filterlog` service. Expanding it to see more details, you should see `pfsense, multiple_blocks` listed under `rule.groups`. This means alerting is working! You can now write your own custom alerts knowing that syslog is working.
