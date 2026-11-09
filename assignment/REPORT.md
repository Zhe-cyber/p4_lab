# Week 5 assignment — Put a limit on the tunnel

Student ID (last 4 digits only): **4982**

Write each section in your own words, about your own runs. Quote the evidence files in
`assignment/evidence/` that support each point. Delete the instructions in `<!-- -->` as you go.

## 1. Analysis: the loop

How did you create the tunnel loop (which entry, on which switch, which commands)?
     What did the switch logs show over time, with numbers from loop-before.txt?
     Then look at the IPv4 path in tunnel.p4: it decrements the TTL. Does that stop an IPv4
     loop in this program? Check the code, and explain what happens when TTL reaches 0.

I started the network with the finished lab program. On s2,tunnel exit 3 was entry handle 2, sending packets to port 3 (h3). I changed it to port 1, which goes back to s1:

     sh ./p4cli s2 "table_modify MyIngress.myTunnel_exact MyIngress.myTunnel_forward 2 1"

Then I sent one packet: `h1 python3 send.py 10.82.0.3 "loop" --tunnel 3`. s1 sends exit 3 to s2, and s2 now sends it back to s1. So the packet goes round and round.

In (`loop-before.txt`). After a few seconds, the logs counted 923 packets on s1 and 922 on s2. Five seconds later the counts were lower (516 and 513). This is because the log file got too big and the switch started a new one. So I used the link counters instead
One packet crossed the link about 1,200 times a second each way. It never stopped. Every crossing was 50 bytes (10,591,890 ÷ 211,837), which is the size of my one packet.
     
ipv4_forward lowers the TTL by 1, 0-1 become 255 and the packet keeps going. The tunnel path is worse, it never changes the TTL, looping TTL= 64 the whole time.

## 2. Design: the hop limit

Your header change and your pipeline change, and the reason for each choice:
     - the field's size, and where in the header you put it (and why there);
     - the starting value send.py uses, and how you chose it;
     - where it's decremented, and what happens when it runs out;
     - what else had to change for every part of the path to agree (parser, deparser, hosts).

**Field:** `hop_limit`, 8 bits. That allows up to 255 hops, the same size as IPv4's TTL.
**Position:** at the end of the tunnel header, after `proto_id` and `dst_id`. The header grows from 4 to 5 bytes. The old fields stay in the same place, so an old switch can still read them (tested in 5a).
**Starting value:** 8. The longest path here is 2 switches, so 8 is enough, with room to
grow. A loop still ends quickly. `--hops` can change it.
**Rule at each switch:** if a tunnelled packet arrives with `hop_limit` 0, drop it.
Otherwise, take 1 away and forward it as before. So a packet sent with 8 is forwarded
8 times and dropped on the 9th switch.
**Other parts:** the parser and deparser did not need changes. They read and write the
whole header, whatever its size. But `send.py` and `receive.py` had to change, so the
hosts use the same 5-byte header as the switches.

## 3. Implementation

Summarise what you changed in tunnel.p4, send.py and receive.py. The pull request shows the code itself: here, explain anything a reviewer might not see from the diff alone.

In tunnel.p4 I added 'hop_limit' to the header, drop at 0, otherwise substract 1 and use 'myTunnel_exact'. The ipv4 path did not change.

In send.py I added 'hop_limit' to the header (default 8) and a '--hops' option. It only accepts 0-255, because the field is 8 bits.

In receive.py I use the same header, and it now prints the 'hop_limit' that arrives.

I also saved tunnel_v1.p4 as a copy of the finished lab program, used as the old switch in 5a question.

## 4. Evidence: it works

Show, with your evidence files:
     - tunnelled delivery to h2 and h3, and the hop limit the receivers saw (delivery.txt);
     - plain IPv4 still works (delivery.txt);
     - the same loop as in section 1 now ends, and after how many passes — does that match
       your design? (loop-after.txt)

-(delivery.txt): packets for exit 2 and 3 arrived at h2 and h3 with 'hop_limit=6'. They left with 8, and s1 and s2 each took 1.
-(delivery.txt): 'h1 ping -c 2 h3' got 2 replies, 0% loss.
-(loop-after.txt): I made the same loop on a fresh network
before sending: s1 packets:0, s2 packets:0
After: s1 packets:5, s2 packets:4
5s later: s1 packets:5, s2 packets:4

This match my design. s1 got the packet with 8,6,4,2 and 0 (dropped at 0). s2 got it with 7,5,3,and 1. That is 9 passes, then it stopped. On the link, each direction went up by 4 packets and 204 bytes. 204/5=51 bytes, which is the old 50 bytes plus 1 hop byte. 

## 5. Evaluation

a) The mixed rollout (mixed.txt): what  did you predict before running s1 with your new
program and s2 with tunnel_v1.p4? What actually happened to delivery, and to the loop?
Explain every difference from your prediction, byte by byte if you need to.

**Prediction before test:**
s1 sends the new 5-byte tunnel header: proto_id (2 bytes), dst_id(2 bytes) and hop_limit (1 byte). s2 still runs the old program, so it only knows the 4-byte header.

Delivery: I put hop_limit at the end of the header, so the old s2 still finds proto_id and dst_id in the same place. I expect s2 to read dst_id=2 correctly and forward the packet out of the port 2 to h2, so the forwarding should still work.

But s2 think the IPv4 header start strainght after the 4byte tunnel header, so it will read my hop_limit byte as the first byte. s2 does not change IPv4 field on the tunnel path, but the program still recalculates the ipv4 checksum for any valid ipv4 header. So if checksum written in wrong position means it will damage the real header. Causing h2 might not accept the packet even though it was forwarded correctly.

For the loop, s2 does not know about the hop_limit so it will not decrease it. Only s1 will count down.

**What happened**(mixed.txt):
For delivery, the old s2 read 4 tunnel bytes ("Bytes parsed: 18"). It read proto_id
(0800) and dst_id (2) correctly and sent the packet to port 2 ("Transmitting packet of
size 52 out of port 2"). But h2 printed nothing. s2 read the IPv4 header starting 1 byte too early, at my hop byte. Then it
updated the checksum ("Updating checksum 'cksum'"). Because it was 1 byte out of place, it wrote over the real IPv4 protocol byte. UDP (0x11) became 0x0d. The packet reached h2, but it was no longer UDP, so receive.py ignored it.

For the loop, s1 handled the packet 9 times (1>10) and s2 8 times (1>9):17 passes,
then it stopped. With two new switches it took 9 passes, so this took about twice as
long.

**Compared with my prediction:** both parts were right. Forwarding worked, but the
checksum update damaged the packet, so h2 did not accept it. The damage was one byte, the protocol field. The loop still ended, because only s1 counted down. This shows that an old switch being able to read the fields is not enough: a switch that misreads a header can still change bytes in it.

b) Propose a safe way to roll out a header change across a network of switches that can't all be upgraded at once. Compare at least two options (for example a version field, a new EtherType, or ordering fields so old switches still parse them).

Option 1, we can add the field at the end. Old switches still forward the packets but as 5a shows, they can quietly damage them. That is the worst kind of failure, becasue it is hard to notice. Option 2, we can use a new RtherType (for example 0x1213 for the new header). Old switches do not know it, so they drop it cleanly and change nothing. Option 3, use a version field. Switches check the version before reading the rest. This only works if the old header already had a version field.

My proposal is to use a new Ethertype. First, upgrade every switch so it understands both the old and the new header. Only then let hosts start sending the new one. Keep the old one working until all senders have moved over.


c) Loop protection in the data plane (this hop limit) versus the control plane (Week 3's
check_paths()): what can each catch that the other can't? Why do real networks use both?

Control plane checks the table entries before they are installed, so it can stop a loop from being created. But it cannot see a change made straight on a switch (like my `table_modify`), short loops while tables are being updated, or bugs in a switch.
Data plane stops any loop while it is happening, whatever caused it. But it does not stop the loop from being created, does not say why it happened, and the packet is still lost.

Real networks use both. The control plane prevents most loops. The data plane limits the damage from the ones that get through.

     
d) What would this change cost on a hardware switch running at line rate?

The cost is small. Each packet gets 1 extra byte. Each switch does one comparison and one
subtraction per packet. No new tables or memory are needed. So a hardware switch can still run at full speed. One small issue: a 5-byte header is an odd size, and some hardware works better with headers that are a multiple of 2 or 4 bytes.


## AI use

This assignment is Level 3 (Open): AI use is allowed if you declare it. State which tools you used (or "None") and what each did — for example "ChatGPT explained the bmv2 log format; the first draft of the hop-limit check came from Copilot, which I then changed and tested". Be ready to explain any part of your work yourself.

I used Claude to help plan the hop-limit design, write the code changes for
`tunnel.p4`, `send.py` and `receive.py`, give me the commands to collect evidence, explain the bmv2 logs and the mixed-rollout result. I ran every test myself in my Codespace and checked the results. The I prepared this report.

## Evidence files

| File | What it shows |
|---|---|
| `evidence/loop-before.txt` | The loop with your finished lab program: pass counts growing |
| `evidence/delivery.txt` | Tunnelled delivery with the hop limit, and a plain ping |
| `evidence/loop-after.txt` | The same loop with your hop limit: pass counts that stop |
| `evidence/mixed.txt` | The mixed rollout: s1 new, s2 old |
