# Week 5 assignment — Put a limit on the tunnel

Student ID (last 4 digits only): **4588**

Write each section in your own words, about your own runs. Quote the evidence files in
`assignment/evidence/` that support each point. Delete the instructions in `<!-- -->` as you go.

## 1. Analysis: the loop
The tunnel loop is created by modifying the third entry (2 when 0-indexed) on switch S2 using the command `sh ./p4cli s2 "table_modify MyIngress.myTunnel_exact MyIngress.myTunnel_forward 2 1"`.
The switch logs in `assignment/evidence/loop-before.txt` show that the number of packets processed keeps increasing due to the loop. (loop-before.txt may show that it appears to decrease but this is because a new file was created as the original file has become too large for the OS.)
It does not stop the loop from happening because the program does not check if IPv4 TTL is more than 0. When the TTL reaches 0, the number will become negative and the loop continues.

## 2. Design: the hop limit

To prevent loops, I created a new field called `ttl` in myTunnel header. The size of the header is 8 bits, because I believe a maximum hop limit of 255 is enough as in IPv4. I placed it after proto_id and dst_id so that the previous fields are not affected.
The starting value send.py uses is 64 because I believe that 64 hops is more than enough to identify loops. This TTL is used by Linux IPv4 packet.
It's decremented in ingress section of the P4 code. When the TTL reaches 0, it will be dropped by the switch.
The switches' parser and the hosts' receive.py and send.py have to be modified to understand the existence of the 8 bit field, TTL.

## 3. Implementation

In tunnel.p4, I changed the myTunnel headers to include the 8-bit ttl header. In the ingress section, I add a check for TTL is a positive number and drops the packet once it reaches 0. In send.py, I added the new 8 bit header with a value of 64. 

## 4. Evidence: it works

<!-- Show, with your evidence files:
     - tunnelled delivery to h2 and h3, and the hop limit the receivers saw (delivery.txt);
     - plain IPv4 still works (delivery.txt);
     - the same loop as in section 1 now ends, and after how many passes — does that match
       your design? (loop-after.txt) -->
The proof that it works is placed in `assignment/evidence/delivery.txt` and `assignment/evidence/loop-after.txt`
In the tunnelled delivery to h2 and h3, the TTL of IPv4 did not change. This is because only the TTL field of MyTunnel header was decremented.
Plain IPv4 still works as shown in the pings in `assignment/evidence/delivery.txt`.
The same loop in section 1 now ends. We can see that S1 received the packet 33 times while S2 received it 32 times. This means that the packets was sent between both switches 32 times and during each egress, the TTL was decremented. This is shown in  `assignment/evidence/loop-after.txt`.

## 5. Evaluation

a) Prediction: First of all, the byte will be out-of-order, especially the IP header. myTunnel TTL field is interpreted as the Version and ihl fields. In a normal delivery scenario, S2 is able to route the packet out of the correct packet and the end host will accept it. In a loop scenario, the TTL will only be decremented by S1, so the loop takes longer to end.
After actual testing without loop, I can verify that S2 forwarded the packet out of port 3 as expected. H3 also received the packet unmodified from S1.
Actual testing with loop shows that only S1 is reducing the TTL in the myTunnel header. Thus, the loop lasts twice as long.
Both are verified in `assignment/evidence/mixed.txt`.

b) There are two possible ways to roll out a header safely: Creating a new header that specifies the revision of the protocol. We can configure the switch to drop the packet safely and notify the sender to use a lower revision protocol. This ensures that none of the packet silently fails. We can also configure the protocol to negotiate the parameters between two switches dynamically. A disadvantage of the method could be that the P4 will be longer due to the multiple versions supported. Another way is to add the new fields at the end of the headers. This ensures that older switches would parse and send the packet correctly as the new field would just be understood as a payload.

c) Loop Protection in the data plane is great as a last resort to end loops in a network. Loop Protection in the control plane prevents loops from being introduced in the first place. It reduces the manpower and time required to debug and find the source of loops in the network. In a ideal scenario, we would not want loops to happen in the first place. Thus in a real network, both should be used with data plane loop protection as the last resort only.

d) In the first scenario (drop if version number is unsupported), there would be a slight increase in processing time and more burden on the hardware as the switch must match the version of protocol supported. In the second scenario (adding new fields at the end of header), there should be minimal impact on the switch.

## AI use

I use Gemini to explain P4 and the v1model, in addition to finding reading resources. However, all modifications to code and report were performed manually without any AI.

## Evidence files

| File | What it shows |
|---|---|
| `evidence/loop-before.txt` | The loop with your finished lab program: pass counts growing |
| `evidence/delivery.txt` | Tunnelled delivery with the hop limit, and a plain ping |
| `evidence/loop-after.txt` | The same loop with your hop limit: pass counts that stop |
| `evidence/mixed.txt` | The mixed rollout: s1 new, s2 old |
