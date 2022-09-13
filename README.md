# Proxy_Egress_Persistence

## Introduction

For the purposes of this research it is assumed that one has already compromised a workstation in a network and has privilege escalated/obtained code execution as local SYSTEM; in addition, CobaltStrike will be the C2 this research is based around however several other C2's have the same/similar functionality that would work just as well. 

I recently encountered an environment in which a web proxy (that could NOT be bypassed and traffic routed directly to the internet) was being used that added some extra wrinkles to the post-exploitation game.  The proxy required network credentials in order to authenticate to and traverse, as well as certain registry keys being set for the user in order to direct traffic to the proxy in the first place.  With a user-context beacon this was of course no concern, as both of these conditions were naturally fulfilled by nature of running the beacon as a user; however it presented a new challenge when it came to talking about high-privileged persistence and/or remote access.  It is suspicious/not desirable to have a Domain Admin or a Workstation Admin or any other flavour of privileged user account calling out via HTTPS given potential GPO's that limit (or try to limit) such behaviour, and the local SYSTEM account on a given workstation has neither credentials to authenticate to the proxy with, nor the registry settings required to utilize the proxy.  Both of these issues can be overcome, but the real question becomes how to do so intelligently.  

![image](https://user-images.githubusercontent.com/91164728/188766744-a048781a-f522-422e-bef0-cafa3bda6cf0.png)

The wishlist for the eventual product to address this problem is as follows:

1. Obtain an HTTPS beacon that is able to traverse the proxy
2. Maintain privileged access to the workstation 
3. Maintain HTTPS beacon access to workstation regardless of whether a user is logged in (this one was not achieved but the theory will be covered)

## Maintaining Privileged Access 

Regardless of how we eventually end up egressing through the proxy, being that code execution as local SYSTEM has been obtained on the compromised host we really want to maintain that level of access.  The easy answer of "just spawn a new HTTPS beacon as SYSTEM" of course doesn't apply here, as our inability to do so is the entire basis of this research.

CobaltStrike (as well as most major C2's) support a variety of different protocols for C2 communications, SMB being a popular choice for intra-network communications between devices. It is far preferable to have one (or maybe two for redundancy) actual network egress points, with other compromised machines communicating out to the C2 infrastructure via intranet SMB connections to the egress host(s).  Computer role also becomes a consideration here; user workstations regularly initiate connections to the internet, domain controllers and file servers do not.

A simple way around the issue we face is rather than spawning an HTTPS beacon as SYSTEM, we can spawn an SMB one.  This will not provide remote access to the compromised host, but it will give us a way to continue to execute code as SYSTEM by linking the spawned SMB beacon to the HTTPS beacon we end up spawning. 

## HTTPS Egress Beacons

As previously mentioned, domain users naturally have the ability to communicate with and traverse the web proxy.  While it is possible to edit the registry settings for the SYSTEM account in order to configure the proxy, and it may even be possible to utilize another account's network credentials (if we had either a TGT or plain text creds) with the SYSTEM account, the path of least resistence is to just utilize a domain user's context in order to run the HTTPS egress beacon. The biggest issue with doing so is that unless we have the plain text credentials for a user (in which case we could use the [LogonUser](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-logonusera) windows API) we are forced to spawn the HTTPS beacon either by running in the context of that user, or by stealing their token from an active process in their session.  

I always like to make my tools/solutions as modular and situation-agnostic as possible; to this end, having to have recovered plain text credentials for a user account is not a viable solution. Given that we also want to spawn and maintain a SYSTEM SMB beacon, the eventual product of this work will need to run in an elevated context so the token stealing option will likely be the easiest to work with. I am going to assume familiarity with the concept of token stealing and will not delve deeply into it here, but bottom line if we are local SYSTEM and there is a user account logged into the machine, we can steal the token of that user and create new processes in their context. Doing so in this case will provide the HTTPS beacon with both the HKCU settings required for the process to locate the proxy, and the network credentials required to authenticate to it.  

The major, glaring issue with the course outlined thus far is that should the user whose token we have stolen log off, all of their processes (including our HTTPS beacon) will exit and we will lose the ability to communicate with the SYSTEM SMB beacon (although it will remain running on the machine, as it exists in session 0).  Requiring a user be logged into the machine in order to have remote access certainly isn't ideal, which led me to spend some time exploring alternative solutions.  An interesting idea that I had and spent some time exploring was trying to spawn a HTTPS beacon as a domain user using a stolen token, in session 0.  Session 0 is reserved for system processes, purposefully isolated from each user's logon session (session 1, session 2, etc).  

## A Quick Note on Running User-Level Processes in Session 0

I did find a way to accomplish spawning a process as a user in session 0 and for a moment thought I was onto something pretty cool; while from a detection standpoint having a user-level process running in session 0 is super weird and suspicious, it opened the door to keeping a process running as a user alive even after that user had logged off/exited their logon session.  The technical details of doing so involved following the traditional token-stealing API path of OpenProcess -> OpenProcessToken -> DuplicateTokenEx with an added call to SetTokenInformation, in which the TokenSessionId value was changed from the user's original session ID to 0, the system session. While this did succeed in spawning an HTTPs beacon as a domain user in session 0, and that beacon process staying alive even after the use had logged off, the network credentials associated with the process were invalidated because the user closed their logon session.  [Will Burgess over at Elastic](https://www.elastic.co/blog/introduction-to-windows-tokens-for-security-practitioners) published a really good article about Windows tokens and the background of the issue I ran into here which I encourage the curious reader to take a look at, as he goes into great technical detail which might prove useful to you in a future situation.  

This major setback led me to do a lot of research into windows authentication and read a little bit further into the actual technical details of things like pass the hash, pass the ticket, and other windows authentication abuse.  After consulting the source code of [Rubeus](https://github.com/GhostPack/Rubeus) I think there might be a path forward along the lines of "Steal a user's TGT and pass that in order to create an HTTPS beacon which (maybe?) would be able to traverse the proxy even after the user has logged off", but there are then additional considerations like that the default lifetime of a TGT is 10 hours, after which it must be renewed.  The line of thought really becomes pretty burdonsome trying to figure out how such a technique would be sustainable in different situations and environments, not to mention the mammmoth task of actually writing the code to accomplish all of it (even though I would shamelessly rip a lot of it from Rubeus). 

## A Path Foward

With what I would have considered the cleanest solution (user-level process in session 0) off the table, I had to consider how 

