---
title: "802.11 Data Units and Framing"
slug: "802-11-data-units-and-framing"
date: "2019-07-24T01:09:53"
categories: ["Uncategorized"]
tags: ["cwna", "notes", "wifi"]
---
Some notes as I study for the CWNA exam.

<!--more-->

**PLCP – Physical Layer *Convergence* Protocol**Service within the PHY layer that receives (or sends) data from the upper layers (2-7). Interaction point between the PHY (Layer 1) and MAC (Layer 2).

**PPDU – PLCP Protocol Data Unit**What is actually transmitted (or received) by the physical medium (in this case, RF symbols)

**PSDU – PLCP Service Data Unit**Same exact thing as the MPDU, but from the context of the PHY layer. Information is added to this to become a PPDU.

**MPDU – MAC Protocol Data Unit**MSDU + additional framing to be handed off to the Physical Layer Convergence Protocol

**MSDU – MAC Service Data Unit**  
Carries upper layer (3-7) data. Same idea as the payload of a Layer 2 Ethernet frame

802.11n added a few more types:

**A-MSDU – Aggregated MSDU**Multiple MSDUs within a single MPDU. Allows for more “payload” data to be transmitted within a single “frame”. Think of TCP windows where ACK overhead can be reduced by sending multiple data packets and then a single ACK. ONLY MSDUs OF THE SAME PRIORITY VALUE MAY BE AGGREGATED! Is a single encrypted payload.

**A-MPDU – Aggregated MPDU**Multiple MPDUs within a single PPDU. Each MPDU within the A-MPDU is encrypted separately. Each MPDU has its own MAC Header. 802.11ac ONLY uses A-MPDUs and therefore requires Block ACKs.

**PHY Preamble** – the first part of the RF transmission. Used to synchronize the receivers to the transmitter. Originally called a SYNC preamble, now called a Training Field as it contains other information than just oscillator sync (HT mode, greenfield in use)

## **Interframe Spaces**

**DIFS – DCF Interframe Space**A station wanting to transmit on the medium is required to wait for a DIFS. If the medium is idle for that DIFS, then it can assume the medium is free. Length is PHY-specific. DIFS = SIFS + (2 \* Slot time)

**SIFS – Short Interframe Space**Used with ACK frames to reduce the amount of time needed to complete a transmission. Pre-802.11n, these were the shortest spaces. “SIFS time consists of the delay in receiver RF, PLCP delay and the MAC processing delay, which depends on the physical layer used.”

**AIFS – Arbitration Interframe Space**Used to “prioritize” classes of traffic. Higher priority traffic uses a shorter AIFS than lower priority traffic. (Voice traffic uses a shorter AIFS than Best Effort traffic.)

**EIFS – Extended Interframe Space**A failsafe that is used when a station hears a corrupted frame. (The frame is likely just fine, but this STA just didn’t hear it properly.) The STA then waits an EIFS before trying to contend for the medium, with the idea being that before that occurs another frame (perhaps the ACK from the actual recipient of the corrupted frame) is heard properly and allows the STA to reset it’s NAV timer.

And then, not on the CWNA exam, are:

**PIFS – PCF Interframe Space (Point Coordination Function)**Generally not used, except for DFS events and Channel Switch Announcements. Shorter than a DIFS, used by the AP only so that the AP can attempt to maintain control of the medium.

**RIFS – Reduced Interframe Space**The shortest space. Used in 802.11n for transmission bursting, but abandoned in 802.11ac

## **Other important timing concepts**

**Slot Time** – the time it takes for the receiver to detect a frame and for the RF to propagate through the environment. Dependent on the physical medium. 802.11g introduced Short Slot Times, which allows for the reduction of the slot time for 2.4GHz RF to 9us from 20us. 5GHz RF uses a slot time of 9us and (currently) has no shorter slot time.

**Guard Interval** – Period of time between RF symbols. Allows for multipath reflections (delay spread) to attenuate before sending the next symbol. 802.11a and 802.11g (OFDM / ERP) use a GI of 800ns. 802.11n (HT) and 802.11ac (VHT) allow for a short GI of 400ns.
