# Week 5 assignment — Put a limit on the tunnel

Student ID (last 4 digits only): **4588**

Write each section in your own words, about your own runs. Quote the evidence files in
`assignment/evidence/` that support each point. Delete the instructions in `<!-- -->` as you go.

## 1. Analysis: the loop

<!-- How did you create the tunnel loop (which entry, on which switch, which commands)?
     What did the switch logs show over time, with numbers from loop-before.txt?
     Then look at the IPv4 path in tunnel.p4: it decrements the TTL. Does that stop an IPv4
     loop in this program? Check the code, and explain what happens when TTL reaches 0. -->
The tunnel loop is created by modify the third entry (2 when 0-indexed) on switch S2 using the command `sh ./p4cli s2 "table_modify MyIngress.myTunnel_exact MyIngress.myTunnel_forward 2 1"`.
The switch logs show that the number of packets processed keeps increasing due to the loop.
It does not stop the loop from happening because the program does not check if IPv4 TTL is more than 0. When the TTL reaches 0, the number will become negative.

## 2. Design: the hop limit

<!-- Your header change and your pipeline change, and the reason for each choice:
     - the field's size, and where in the header you put it (and why there);
     - the starting value send.py uses, and how you chose it;
     - where it's decremented, and what happens when it runs out;
     - what else had to change for every part of the path to agree (parser, deparser, hosts). -->
To prevent loops, I created a new field called `ttl` in myTunnel header. I placed it after proto_id and dst_id so that the previous fields are not affected.
The starting value send.py uses is 64 because I believe that 64 hops is more than enough to determine loops. This TTL is used by Linux IPv4 packet.
It's decremented in ingress section of the P4 code. When the TTL reaches 0, it will be dropped by the switch.
The parser has to be modified to understand the existence of the 8 bit field, TTL.

## 3. Implementation

<!-- Summarise what you changed in tunnel.p4, send.py and receive.py. The pull request shows
     the code itself: here, explain anything a reviewer might not see from the diff alone. -->
In tunnel.p4, I changed the myTunnel headers to include the 8-bit ttl field. In send.py, I added the new 8 bit header with a value of 8 bits. 

## 4. Evidence: it works

<!-- Show, with your evidence files:
     - tunnelled delivery to h2 and h3, and the hop limit the receivers saw (delivery.txt);
     - plain IPv4 still works (delivery.txt);
     - the same loop as in section 1 now ends, and after how many passes — does that match
       your design? (loop-after.txt) -->

In the tunnelled delivery to h2 and h3, the TTL of IPv4 did not change. This is because the TTL field of MyTunnel header changed.
Plain IPv4 still works as shown in the pings.
The same loop in section 1 now ends. We can see that S1 received the packet 33 times while S2 received it 32 times. This makes 

## 5. Evaluation

<!-- a) The mixed rollout (mixed.txt): what did you predict before running s1 with your new
        program and s2 with tunnel_v1.p4? What actually happened to delivery, and to the loop?
        Explain every difference from your prediction, byte by byte if you need to.
     b) Propose a safe way to roll out a header change across a network of switches that can't
        all be upgraded at once. Compare at least two options (for example a version field, a
        new EtherType, or ordering fields so old switches still parse them).
     c) Loop protection in the data plane (this hop limit) versus the control plane (Week 3's
        check_paths()): what can each catch that the other can't? Why do real networks use both?
     d) What would this change cost on a hardware switch running at line rate? -->

First of all, the byte will be out-of-order, especially the IP header. myTunnel TTL field is interpreted as the Version and ihl fields. In a normal delivery scenario, S2 is able to route the packet out of the correct packet and the end host will accept it. In a loop scenario, the TTL will only be decremented by S1, so the loop takes longer to end.
After actual testing, I can verify that S2 forwarded the packet out of port 3 as expected. H3 also received the packet unmodified from S1.


## AI use

<!-- This assignment is Level 3 (Open): AI use is allowed if you declare it. State which tools you
     used (or "None") and what each did — for example "ChatGPT explained the bmv2 log format;
     the first draft of the hop-limit check came from Copilot, which I then changed and tested".
     Be ready to explain any part of your work yourself. -->

## Evidence files

| File | What it shows |
|---|---|
| `evidence/loop-before.txt` | The loop with your finished lab program: pass counts growing |
| `evidence/delivery.txt` | Tunnelled delivery with the hop limit, and a plain ping |
| `evidence/loop-after.txt` | The same loop with your hop limit: pass counts that stop |
| `evidence/mixed.txt` | The mixed rollout: s1 new, s2 old |
