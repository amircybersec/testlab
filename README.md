# Test Lab
In this repo, I plan to include several network level attack prototypes that are intended to disrupt data flow. This is experimental and research effort and the code here must be used with care and not in any malicious intent. The goal of the experiment is to emulate a hostile network environment that is typically observed in countries with censorship (China, Iran , etc) for testing and measuring resiliency of solutions against such attacks.

## TCP RESET Attack

This type of attack is well observed in China and Iran and is deployed to disrupt flow of traffic. In this attack model, a third device on the network that can sniff the traffic between the client & server, can forge a new TCP packet with RESET flag and send it as a fake response coming from the server to end the connection. This tactic is used by firewalls mainly to first allow the traffic to go through, analyze its content, and block if certain condistions are met. This method also has less colateral damage since blocking destination IP addresses altogether may block access to other services. 

These two documents [1](https://robertheaton.com/2020/04/27/how-does-a-tcp-reset-attack-work/),[2](https://squidarth.com/article/networking/2020/05/03/tcp-resets.html) provide great insights and details, that I highly recommend reading them if you want to get the full picture. In those documents, a fully local test is done using loopback adaptor and netcat (nc) to do the dial function. 

### First experiment

The first experiment is setup to break attempts to do domain name resolution over TCP (TCP RESET attack only applied to TCP connections). The following comand performs a name resolution using the resolver 8.8.8.8. We setup the experiment such that the attacker (/reset_attack/main.py) sniffs the traffic on the device interface, and whenever a packet is sent to 8.8.8.8 it will craft a new packet pretending to be from 8.8.8.8 and sending it to dig client.  

```
dig @8.8.8.8 google.com +tcp
```

The following wireshark capture shows the DNS resolution over TCP before injecting reset packets:

![image](https://github.com/amircybersec/testlab/assets/117060873/3da686e6-0ed0-422c-88d4-f7566d7e187a)


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

