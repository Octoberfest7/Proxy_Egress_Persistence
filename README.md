# Proxy_Egress_Persistence

## Introduction

For the purposes of this research it is assumed that one has already compromised a workstation in a network and has privilege escalated/obtained code execution as local SYSTEM; in addition, CobaltStrike will be the C2 this research is based around however several other C2's have the same/similar functionality that would work just as well. 

I recently encountered an environment in which a web proxy was being used that added some extra wrinkles to the post-exploitation game.  The proxy required network credentials in order to authenticate to and traverse, as well as certain registry keys being set for the user in order to direct traffic to the proxy in the first place.  With a user-context beacon this was of course no concern, as both of these conditions were naturally fulfilled by nature of running the beacon as a user; however it presented a new challenge when it came to talking about high-privileged persistence and/or remote access.  It is suspicious/not desirable to have a Domain Admin or a Workstation Admin or any other flavour of privileged user account calling out via HTTPS given potential GPO's that limit (or try to limit) such behaviour, and the local SYSTEM account on a given workstation has neither credentials to authenticate to the proxy with, nor the registry settings required to utilize the proxy.  Both of these issues can be overcome, but the real question becomes how to do so intelligently.  

![image](https://user-images.githubusercontent.com/91164728/188766744-a048781a-f522-422e-bef0-cafa3bda6cf0.png)

The wishlist for the eventual product to address this problem is as follows:

1. Obtain an HTTPS beacon that is able to traverse the proxy
2. Maintain privileged access to the workstation 
3. Maintain HTTPS beacon access to workstation regardless of whether a user is logged in (this one was not achieved but the theory will be covered)

## Maintaining Privileged Access 

Regardless of how we eventually end up egressing the proxy, being that code execution as local SYSTEM has been obtained on the compromised host we really want to maintain that level of access.  The easy answer of "just spawn a new HTTPS beacon as SYSTEM" of course doesn't apply here, as our inability to do so is the entire basis of this research. A simple way around this is rather than spawning an HTTPS beacon as SYSTEM, we can spawn an SMB one.  CobaltStrike (as well as most major C2's) support a variety of different protocols for C2 communications, SMB being a popular choice for intra-network communications between devices.  It is far preferable to have one (or maybe two for redundancy) actual network egress points, with other compromised machines communicating out to the C2 infrastructure via intranet SMB connections to the egress hosts.  Computer role also becomes a consideration here; user workstations regularly initiate connections to the internet, domain controllers and file servers do not. For our purposes, instead of using an SMB beacon to run commands on another host in the network, we will spawn an SMB beacon on the egress host; the one that is also communicating out to the internet.  

