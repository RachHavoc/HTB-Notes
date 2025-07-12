Simplest TCP Port Scan relies on three-way TCP handshake. 

Host sends TCP SYN -> Open Destination Port responds with SYN-ACK -> Host sends ACK

If handshake completes, port is open.

Netcat performing TCP port scan 

```
kali@kali:~$ nc -nvv -w 1 -z 192.168.50.152 3388-3390
(UNKNOWN) [192.168.50.152] 3390 (?) : Connection refused
(UNKNOWN) [192.168.50.152] 3389 (ms-wbt-server) open
(UNKNOWN) [192.168.50.152] 3388 (?) : Connection refused
 sent 0, rcvd 0
```

Netcat performing UDP port scan 

```
kali@kali:~$ nc -nv -u -z -w 1 192.168.50.149 120-123
(UNKNOWN) [192.168.50.149] 123 (ntp) open
```

An empty UDP packet is sent. 
If UDP port is open, packet passed to application layer. Response depends on how app layer is programmed. This can be a false positive if there is a firewall.

If UDP port is closed, target responds with ICMP port unreachable. 