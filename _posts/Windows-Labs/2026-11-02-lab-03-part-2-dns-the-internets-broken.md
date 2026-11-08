---
layout: post
title: "Lab 03 Part 2: DNS – \"The Internet's Broken\""
date: 2026-11-02
---

## The ticket

"The internet's broken." It's one of the most common calls a help desk gets, and it's almost never the whole internet. For this lab I broke DNS on purpose in a Windows 11 VM, then fixed it like a real ticket. It didn't go to plan, and I learnt more because of that.

**My analogy:** the internet is a party across town. Your PC knows party *addresses* (IP addresses), but people only ever give it party *names* (google.com). DNS is the drunk mate at the front door who tells you which house it is. If he's passed out, you can still drive to any address you know, but you'll never find the party by name. The user thinks the whole internet is broken. Really, the drunk mate has just stopped answering.

## Lab setup

- Windows 11 VM in VirtualBox, using VirtualBox's default NAT network
- Default gateway `10.0.2.2` and DNS server `10.0.2.3`: VirtualBox's built-in pretend router and DNS
- Tools: Settings, Command Prompt (`ipconfig`, `ping`, `nslookup`) and Edge

## Breaking it (harder than it sounds)

In **Settings > Network & internet > Ethernet**, I set **DNS server assignment** to Manual and pointed IPv4 DNS at `10.255.255.1`, a private address nothing answers on. Then I opened Edge, and every website still loaded.

**Twist 1: the cache.** Windows and the browser remember addresses they've already looked up. It's like having last week's party addresses scribbled on your hand. I ran `ipconfig /flushdns`, closed Edge and tried a new site. It still loaded.

**Twist 2: the hidden second DNS server.** `nslookup abc.net.au` timed out against `10.255.255.1`, so my change had worked. But `ping abc.net.au` still replied. Running `ipconfig /all` again showed why: under **DNS Servers** there was an IPv6 DNS server sitting underneath my IPv4 one.

- `nslookup` rings only the first DNS server directly. No answer means it just times out.
- `ping`, Edge and everything else go through the Windows resolver. That's like a receptionist who quietly tries every number in the contact book when the first doesn't answer. She fell back to the IPv6 server, so the user would never have known anything was wrong.

I only broke it properly once I also set IPv6 DNS to `2001:db8::1`, an address reserved for documentation that nothing ever answers on. After another flush, `ping abc.net.au` sat there and failed.

## Diagnosing it like a real ticket

1. `ping abc.net.au` gives a long pause, then fails. A long wait followed by failure usually means something is **timing out**, not refusing.
2. `ping 8.8.8.8` replies. By IP address, the adapter, the gateway and the internet are all fine.
3. Name fails but IP works, so the problem is DNS. The car runs fine. It's only the drunk mate at the door who's out cold.
4. `ipconfig /all` lists **every** DNS server, not just the first, which tells you what to check.
5. `nslookup` shows exactly which DNS server is (or isn't) answering.

## Fixing it

1. **Settings > Network & internet > Ethernet > DNS server assignment > Edit**, then switch from Manual back to **Automatic (DHCP)**.
2. `ipconfig /flushdns`
3. Confirm with `ping abc.net.au` and `nslookup abc.net.au`. The Server line was back to `10.0.2.3`.

One thing to watch: I only changed the **DNS server** setting, never the **IP address** setting. The VM kept getting its IP automatically the whole time. Windows keeps these as two separate settings, and changing the wrong one can turn a DNS problem into "no network at all".

## Lessons learned

- **Flush the cache after any DNS change.** I forgot this in a previous lab, and the fix looked like it hadn't worked until I flushed.
- **Check every DNS server entry, IPv4 and IPv6.** Windows falls back quietly, so one broken server can hide behind a working one.
- **`nslookup` and `ping` can disagree.** `nslookup` asks one server directly, while `ping` goes through the Windows resolver and its fallbacks.
- **A long pause, then failure, means a timeout.** A refusal comes back fast.
- **Ping by IP, then by name.** If IP works and name doesn't, it's DNS.
- **DNS settings and IP settings are separate.** Change only the one you mean to.

If an interviewer asks me to walk them through "a user says the internet's down", this is the lab I'll talk about.
