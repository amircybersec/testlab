# Test Lab
In this repo, I plan to include several network level attack prototypes that are intended to disrupt data flow. This is experimental and research effort and the code here must be used with care and not in any malicious intent. The goal of the experiment is to emulate a hostile network environment that is typically observed in countries with censorship (China, Iran , etc) for testing and measuring resiliency of solutions against such attacks.

## TCP RESET Attack

This type of attack is well observed in China and Iran and is typically deployed to disrupt flow of traffic. 

### First experiment
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

