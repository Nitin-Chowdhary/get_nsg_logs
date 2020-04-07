# Reading NSG logs from Blob Storage

You might have set your Azure Vnet, with some NSGs. You start rolling apps, to the point where you have many VMs, and many NSGs. Somebody makes an application upgrade, or installs a new application, but traffic is not flowing through. Which NSG is dropping traffic? Which TCP ports should be opened?

One possibility is using Traffic Analytics. Traffic Analytics is a two-step process:
1. NSG logs are stored in a storage account
2. NSG logs from the storage account are processed (consolidated and enriched with additional information) and made queriable

One of the nicest features of Traffic Analytics is being able to query logs with the KQL (Kusto Query Language). For example, you can use this query to find out the dropped flows in the last 3 hours for IP address 1.2.3.4:

```
AzureNetworkAnalytics_CL
| where TimeGenerated >= ago(1h)
| where SubType_s == "FlowLog"
| where DeniedInFlows_d > 0 or DeniedOutFlows_d > 0
| where SrcIP_s == "1.2.3.4"
| project NSGName=split(NSGList_s, "/")[2],NSGRules_s,DeniedInFlows_d,DeniedOutFlows_d,SrcIP_s,DestIP_s,DestPort_d,L7Protocol_s
```

However, you will notice that there is a time lag, and you will not find the very latest logs in Log Analytics. The original NSG Flow logs are stored in storage account, in JSON format, so you could get those logs using the Azure Storage SDK.

You can use the Python script in this repository (assuming you have the Python SDK for storage installed). You can use different flags, like the --help option to get usage information. If you do not have handy a Python console you can use Azure Cloud Shell and clone this repository:

```
git clone https://github.com/erjosito/get_nsg_logs
```

You might to install the Python module for Azure Storage:

```
 pip install azure-storage --user 
```
Now you can have a look at the different options:

```
$ cd ./get_nsg_logs/
$ python3 ./get_nsg_logs.py --help
usage: get_nsg_logs.py [-h] [--account-name ACCOUNT_NAME] [--display-lb]
                       [--display-allowed]
                       [--display-direction DISPLAY_DIRECTION]
                       [--display-hours DISPLAY_HOURS] [--version VERSION]
                       [--only-non-zero] [--flow-state FLOW_STATE_FILTER]
                       [--ip IP_FILTER] [--port PORT_FILTER]
                       [--nsg-name NSG_NAME_FILTER] [--aggregate] [--verbose]

Get the latest flow logs in a storage account

optional arguments:
  -h, --help            show this help message and exit
  --account-name ACCOUNT_NAME
                        you need to supply an storage account name. You can
                        get a list of your storage accounts with this command:
                        az storage account list -o table
  --display-lb          display or hide flows generated by the Azure LB
                        (default: False)
  --display-allowed     display as well flows allowed by NSGs (default: False)
  --display-direction DISPLAY_DIRECTION
                        display flows only in a specific direction. Can be in,
                        out, or both (default in)
  --display-hours DISPLAY_HOURS
                        How many hours to look back (default: 1)
  --version VERSION     NSG flow log version (1 or 2, default: 1)
  --only-non-zero       display only v2 flows with non-zero packet/byte
                        counters (default: False)
  --flow-state FLOW_STATE_FILTER
                        filter the output to a specific v2 flow type (B/C/E)
  --ip IP_FILTER        filter the output to a specific IP address
  --port PORT_FILTER    filter the output to a specific TCP/UDP port
  --nsg-name NSG_NAME_FILTER
                        filter the output to a specific NSG
  --aggregate           run in verbose mode (default: False)
  --verbose             run in verbose mode (default: False)
  ```

There is something you need to do before being able to access Azure Blob Storage: finding out the Azure Storage Account key. The script will read it from the environtment variable STORAGE_ACCOUNT_KEY, that you can set with this command:

```
export STORAGE_ACCOUNT_KEY=$(az storage account keys list -n your_storage_account_name --query [0].value -o tsv)
```

For example, in order to show dropped and allowed traffic of ingress NSG logs stored in the storage account `ernetworkhubdiag857`, excluding Azure LB probe traffic for the last 6 hours:

```
$ python3 ./get_nsg_logs.py --account-name ernetworkhubdiag857 --display-hours 6 --display-direction in --displayAllowed
2018-09-21T09:49:57.3055150Z NVA-NSG DefaultRule_AllowVnetInBound A I 10.90.15.47 29014 10.139.149.70 23
2018-09-21T09:49:57.3055150Z NVA-NSG DefaultRule_AllowVnetInBound A I 10.90.15.47 29014 10.139.149.70 23
2018-09-21T09:49:57.3055150Z NVA-NSG DefaultRule_AllowVnetInBound A I 10.90.15.47 32069 10.139.149.70 23
2018-09-21T09:49:57.3055150Z NVA-NSG DefaultRule_AllowVnetInBound A I 10.90.15.47 29014 10.139.149.70 23
2018-09-21T09:49:57.3055150Z NVA-NSG DefaultRule_AllowVnetInBound A I 10.90.15.47 29014 10.139.149.70 23
2018-09-21T09:49:57.3055150Z NVA-NSG DefaultRule_AllowVnetInBound A I 10.90.15.47 32069 10.139.149.70 23
```

If you are using the version 2 of the flow log export format, you can show some additional options:

```
$ python3 ./get_nsg_logs.py --account-name fwtestdiag354 --version 2 --display-allowed --display-direction both --only-non-zero --port 80 --display-hours 2 --aggregate
2019-10-16T22:20:16.9434260Z FWTESTVM-NSG DefaultRule_AllowInternetOutBound A O 192.168.1.4 tcp 39884 104.28.19.94 80 E src2dst: 6/419 dst2src: 4/629
2019-10-16T21:32:16.8646597Z FWTESTVM-NSG DefaultRule_AllowInternetOutBound A O 192.168.1.4 tcp 54326 51.137.52.221 80 E src2dst: 17/1787 dst2src: 120/172007
2019-10-16T21:32:16.8646597Z FWTESTVM-NSG DefaultRule_AllowInternetOutBound A O 192.168.1.4 tcp 34680 91.189.88.24 80 E src2dst: 25/1875 dst2src: 66/93416
Totals src2dst -> 48 packets and 4081 bytes
Totals dst2src -> 190 packets and 266052 bytes
```

# Testing the tool

Let us generate a typical HTTP request, we will use the service ifconfig.co:

```
curl -s4 ifconfig.co
```

In a separate screen you can capture the traffic. Here you have it for the example above (captured with tcpdump in a Linux VM in Azure):

```
07:11:16.387239 IP fwtestvm.38264 > 104.28.19.94.http: Flags [S], seq 432244736, win 64240, options [mss 1460,sackOK,TS val 3932777591 ecr 0,nop,wscale 7], length 0
07:11:16.399859 IP 104.28.19.94.http > fwtestvm.38264: Flags [S.], seq 1557095350, ack 432244737, win 29200, options [mss 1400,nop,nop,sackOK,nop,wscale 10], length 0
07:11:16.399890 IP fwtestvm.38264 > 104.28.19.94.http: Flags [.], ack 1, win 502, length 0
07:11:16.399991 IP fwtestvm.38264 > 104.28.19.94.http: Flags [P.], seq 1:76, ack 1, win 502, length 75: HTTP: GET / HTTP/1.1
07:11:16.412831 IP 104.28.19.94.http > fwtestvm.38264: Flags [.], ack 76, win 29, length 0
07:11:16.457021 IP 104.28.19.94.http > fwtestvm.38264: Flags [P.], seq 1:390, ack 76, win 29, length 389: HTTP: HTTP/1.1 200 OK
07:11:16.457045 IP fwtestvm.38264 > 104.28.19.94.http: Flags [.], ack 390, win 501, length 0
07:11:16.457194 IP fwtestvm.38264 > 104.28.19.94.http: Flags [F.], seq 76, ack 390, win 501, length 0
07:11:16.470909 IP 104.28.19.94.http > fwtestvm.38264: Flags [F.], seq 390, ack 77, win 29, length 0
07:11:16.470931 IP fwtestvm.38264 > 104.28.19.94.http: Flags [.], ack 391, win 501, length 0
```

You can see the TCP 3-way handshake (first 3 packets), the GET request (next 2 packets), the HTTP answer (next 2 packets), and the TCP finalization (last 3 packets). 10 packets in total. Let us check our NSG flows:

```
>python ./get_nsg_logs.py --account-name fwtestdiag354 --version 2 --display-allowed --display-direction out --only-non-zero --port 80 --aggregate --display-hours 2 --ip 104.28.19.94 --port 38264
2019-10-17T07:12:17.6127249Z FWTESTVM-NSG DefaultRule_AllowInternetOutBound A O 192.168.1.4 tcp 38264 104.28.19.94 80 E src2dst: 6/419 dst2src: 4/629
Totals src2dst -> 6 packets and 419 bytes
Totals dst2src -> 4 packets and 629 bytes
```

10 packets: Check! All of the packets, including the first SYN and the last ACK were captured in the flow logs. Let us check whether half-open TCP connections are verified too. We will use the tool hping3 (on Ubuntu you can install it with `sudo apt install -y hping3`):

```
$ sudo hping3 -V -S -p 80 104.28.19.94
using eth0, addr: 192.168.1.4, MTU: 1500
HPING 104.28.19.94 (eth0 104.28.19.94): S set, 40 headers + 0 data bytes

len=46 ip=104.28.19.94 ttl=52 DF id=0 tos=0 iplen=44
sport=80 flags=SA seq=0 win=29200 rtt=15.1 ms
seq=554566472 ack=1426251946 sum=cc85 urp=0

len=46 ip=104.28.19.94 ttl=52 DF id=0 tos=0 iplen=44
sport=80 flags=SA seq=1 win=29200 rtt=14.7 ms
seq=3991602591 ack=1322897034 sum=dfce urp=0

len=46 ip=104.28.19.94 ttl=52 DF id=0 tos=0 iplen=44
sport=80 flags=SA seq=2 win=29200 rtt=14.6 ms
seq=2340604942 ack=1706818275 sum=9d27 urp=0

^C
--- 104.28.19.94 hping statistic ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 14.6/14.8/15.1 ms
```

Our capture shows 9 packets (3 for each hping3 SYN message):

```
6:40:11.581669 IP fwtestvm.2300 > 104.28.19.94.http: Flags [S], seq 1426251945, win 512, length 0
06:40:11.593977 IP 104.28.19.94.http > fwtestvm.2300: Flags [S.], seq 554566472, ack 1426251946, win 29200, options [mss 1400], length 0
06:40:11.593997 IP fwtestvm.2300 > 104.28.19.94.http: Flags [R], seq 1426251946, win 0, length 0
06:40:12.582130 IP fwtestvm.2301 > 104.28.19.94.http: Flags [S], seq 1322897033, win 512, length 0
06:40:12.594818 IP 104.28.19.94.http > fwtestvm.2301: Flags [S.], seq 3991602591, ack 1322897034, win 29200, options [mss 1400], length 0
06:40:12.594848 IP fwtestvm.2301 > 104.28.19.94.http: Flags [R], seq 1322897034, win 0, length 006:40:13.582315 IP fwtestvm.2302 > 104.28.19.94.http: Flags [S], seq 1706818274, win 512, length 0
06:40:13.595058 IP 104.28.19.94.http > fwtestvm.2302: Flags [S.], seq 2340604942, ack 1706818275, win 29200, options [mss 1400], length 0
06:40:13.595081 IP fwtestvm.2302 > 104.28.19.94.http: Flags [R], seq 1706818275, win 0, length 0
```

And our NSG flow counters show the new flows (with TCP source ports 2300, 2301 and 2302), with 3 packets for each flow. Note how the state is E:

```
python ./get_nsg_logs.py --account-name fwtestdiag354 --version 2 --display-allowed --display-direction out --only-non-zero --port 80 --aggregate --ip 104.28.19.94
2019-10-17T06:33:17.5758719Z FWTESTVM-NSG DefaultRule_AllowInternetOutBound A O 192.168.1.4 tcp 56228 104.28.19.94 80 C src2dst: 6/420 dst2src: 8/4498
2019-10-17T06:41:17.5858392Z FWTESTVM-NSG DefaultRule_AllowInternetOutBound A O 192.168.1.4 tcp 2301 104.28.19.94 80 E src2dst: 2/108 dst2src: 1/60
2019-10-17T06:41:17.5858392Z FWTESTVM-NSG DefaultRule_AllowInternetOutBound A O 192.168.1.4 tcp 2300 104.28.19.94 80 E src2dst: 2/108 dst2src: 1/60
2019-10-17T06:41:17.5858392Z FWTESTVM-NSG DefaultRule_AllowInternetOutBound A O 192.168.1.4 tcp 2302 104.28.19.94 80 E src2dst: 2/108 dst2src: 1/60
Totals src2dst -> 12 packets and 744 bytes
Totals dst2src -> 11 packets and 4678 bytes
```

# How long do you have to wait?

For new flows the information should be in the logs in around 1 min.
