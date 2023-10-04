# Test Lab
In this repo, I plan to include several network level attack prototypes that are intended to disrupt data flow. This is experimental research effort and the code here must be used with care and not in any malicious intent. The goal of the experiment is to emulate a hostile network environment that is typically observed in countries with censorship (China, Iran , etc) for testing and measuring resiliency of solutions against such attacks.

## TCP RESET Attack

This type of attack is well observed in China and Iran and is deployed to disrupt flow of traffic. In this attack model, a third device on the network that can sniff the traffic between the client & server, forge a new TCP packet with RESET flag, and send it as a fake response coming (from the destination/server) to end the connection. This tactic is used by firewalls to initially allow the traffic to go through, analyze its content, and block it if certain condistions are met. This method also has less colateral damage since blocking destination IP addresses or ports may block access to other services. 

These two documents [1](https://robertheaton.com/2020/04/27/how-does-a-tcp-reset-attack-work/),[2](https://squidarth.com/article/networking/2020/05/03/tcp-resets.html) provide great insights and details, and I highly recommend reading them if you want to get the full picture. In those documents, a fully local test (attacker, client, server on the same computer) is done using loopback adaptor and netcat (nc) to do the dial function. 

### Setting up your lab

I suggest trying to repeat tests and reporduce the results in the experiments in the above documents on your computer. I also suggest going one step further and run your client and server on different machines (either on the same local netwokr or removely in a VPS). Please note to properly configure your firewall rules on your VPS provider, operating system (e.g. `ufw` on Ubuntu). If you run your server on home network and client connects from outside, you also need to setup port forwarding on your router for the listenging port.  

### First experiment

The first experiment is designed to disrupt attempts to perform domain name resolution over TCP (TCP RESET attack only applied to TCP connections). The following command performs a name resolution using the resolver 8.8.8.8. DNS resolution is usually done over UDP but the datagram packets can be carried with TCP payload. The advantage of this test is that you don't need to setup a listening server, which makes the setup a bit easier. 

```
dig @8.8.8.8 google.com +tcp
```

We setup the experiment such that the attacker (`/reset_attack/main.py`): 

  1. Sniffs the traffic on the device interface (e.g. `eth01`)
  2. When a packet is sent to 8.8.8.8, it will craft a new packet (pretending to be from 8.8.8.8) destined to `dig` client 
  3. It will inject the crafted packet into the network interface

The following wireshark capture shows the DNS resolution over TCP before injecting reset packets:

![image](https://github.com/amircybersec/testlab/assets/117060873/3da686e6-0ed0-422c-88d4-f7566d7e187a)

Next, we are going to repeat DNS resolution over TCP and this time, we are going to inject a reset packget.

To reporduce the results, run the python scapy script on your local machine and make sure you set the following parameters correctly in the script

 - the client IP (your interface IP),
 - server IP (8.8.8.8)
 - the server port (53)

Make sure run the script with superuser previlege:

```
sudo PYTHONPATH=$HOME/.local/lib/python3.10/site-packages/ python main.py
```

In the above case, the scapy package is installed at `$HOME/.local/lib/python3.10/site-packages/`. 

If the above script is running, running dig will through "connection reset" error as shown below. Please note that dig retries a few times and aborts after several tries.

```
> dig @8.8.8.8 google.com +tcp
;; Connection to 8.8.8.8#53(8.8.8.8) for google.com failed: timed out.
;; communications error to 8.8.8.8#53: connection reset
;; Connection to 8.8.8.8#53(8.8.8.8) for google.com failed: timed out.
```

### Timing challenges

Once of the most challenging aspect of performing TCP Reset attack is making sure the forged TCP packet arrives at the client before server actual response (ACK) has arrived. This is due to the design of TCP to strictly enforce sequence number of packets with RESET flag. Initially in my tests, all of the reset packets were arriving too late, after the connection was finalized and terminated. It turned out that the server responses were beating me to the punch and arriving sooner. The picture below shows reset packets (shown by red lines) all arrive after the communication is terminated. 

![image](https://github.com/amircybersec/testlab/assets/117060873/a432995f-96b0-4d2e-b59f-9e28547fc2e4)

There are a few observatons here. When I ran the above test, my laptop was on WiFi, and the WiFi access point was connected to Internet with Fiber optics which is known to have very low latency. In otherwords, the connected between client and server (my laptop and google server) was after than my python script. Basically, the sniff, analyze, craft packet and inject operations were in total slower than server response time. 

To work around this, I decided to slowdown my Internet connection and increase latency. I guess there are more precise ways to slowdown a Internet link but my first hunch was to use my celluar data since it is really bad, specially in my room. (I had never been excited by the prospects of a sh*ty service). It turns out that the slow internet connection in this case, can give the attacker an timing edge. The image below the repeat of the same test but this time data traveling through cellular network to Google DNS.

![image](https://github.com/amircybersec/testlab/assets/117060873/4e55068d-98d7-43f1-9968-dfa4318cc593)


```
go run github.com/Jigsaw-Code/outline-sdk/x/examples/outline-connectivity@latest -v -transport="split:1" -proto tcp -resolver 8.8.8.8
```

### Second experiment
```
KEY=ss://examplekey_V0Zi1wb2x5MTMwNTpLeTUyN2duU3FEVFB3R0JpQ1RxUnlT@104.x.x.x:65496/
PREFIX=POST%20
go run github.com/Jigsaw-Code/outline-sdk/x/examples/outline-connectivity@latest -v -transport="$KEY?prefix=$PREFIX" -proto tcp -resolver 8.8.8.8 && echo Prefix "$PREFIX" works!
```

### Attack successful
```
[DEBUG] 2023/10/01 22:21:38.004242 main.go:135: Test error: read: failed to read salt: read tcp 192.x.x.x:33422->104.x.x.x:65496: read: connection reset by peer
{"resolver":"8.8.8.8:53","proto":"tcp","time":"2023-10-02T05:21:37Z","duration_ms":112,"error":{"op":"read","posix_error":"ECONNRESET","msg":"connection reset by peer"}}
exit status 1
```

### Timing challenges:

Reset packages arriving too late after the connection is closed with FIN flag or the SEQ number has already increased. 

