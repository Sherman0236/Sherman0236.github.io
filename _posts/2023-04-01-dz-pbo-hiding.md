---
title: PBO Hiding in DayZ
author: sherman
date: 2023-04-01
categories: [DayZ]
tags: [PBO-Ninja]
---

During the height of Arma 3, the game-hacking black market was dominated by a form of cheat known as a "PBO hider."
A PBO hider allowed users to load SQF scripts without an ounce of coding experience. People could simply
go to their favorite game-hacking website/forum and download or purchase what is essentially an anti-cheat bypass. 
The ease of use, lack of prerequisites, and [public nature](https://www.unknowncheats.me/forum/755921-post1.html) are what attracted so many users to these popular products. 

## WHAT IS A PBO?
The [.PBO file format](https://community.bistudio.com/wiki/PBO_File_Format) first premiered with Operation Flashpoint in the Summer of 2001. A PBO or "packed bank of files" is
similar in purpose to a zip file. It simply serves as a medium for storing and compressing files and folders. Apple's LZSS 
compression can be selectively used on individual files within a PBO. Two types of PBO exist: addon and mission. 
A mission PBO instructs the game on how to construct a playable scenario. An addon PBO is what might be referred to in
other gaming communities as a mod. Many of Bohemia Interactive's games such as DayZ and Arma 3 can be configured to load any PBO file
a user desires. These games also blindly load any PBO file residing in certain directories upon game launch. 
These PBO files hold critical data related to the game's functionality and any additional functionality provided by mods. 

## SECURING ONLINE PLAY
A keen reader might point out that loading any addon (PBO file) in a directory without validation is dangerous, and they are correct. 
However, these addons do not pose any threat to the gameplay integrity of other players because preventative measures exist to 
prevent a client with unrecognized addons from entering secure servers. Upon connecting to a secure multiplayer environment, the client sends 
information about each loaded addon to the game server. Included in this information is the digital signature and hash of the addon's PBO file.
The game server then determines if the client is running without addons that the server requires, with an addon that the server does not run,
or with a modified addon. If any of these are found to be true, then the connection with the client is terminated. 

But how does the server know if the client is running too many/few PBOs or if a PBO is modified? To verify the integrity of the client's game, 
the server checks the list of addons that the client sent against its own list of addons. Once it has been verified that the client is
running the same addons as the server, the integrity of the addons is verified in two steps.
First, a PBO must be signed with a valid digital signature, and the game server must accept this digital signature. 
Secondly, the hash of the client's PBO must match the hash of the PBO on the server. This prevents the game client from connecting 
to a secure multiplayer environment with any PBOs loaded that the game server does not recognize. 

## PBO HIDERS IN ARMA 3
Arma 3 had a simple flaw that allowed for these PBO addons to be ignored by the game and thus loaded in multiplayer. The above-mentioned
list of addons sent to the server was stored in an array of QFBanks in memory called `GFileBanks`. (A QFBank is a class representing a loaded addon.)
PBO hiders had the single purpose of bypassing the server's checks for additional addons. The problem is simple and so is the solution: 
remove the addon containing a payload (usually cheats) from the list of loaded PBO addons that is sent to the server. 
The server validates each PBO without seeing the malicious addon, and allows the user to join. In 2018, Bohemia Interactive released update 1.86 for Arma 3.
Among the [patchnotes for this update](https://dev.arma3.com/post/spotrep-00082), is a conspicuous line which reads:\
`Added: Improvements to multiplayer security systems related to addon loading, encryption, and server checks.`

Bohemia finally took a stand after years of cheaters abusing such a simple integrity check. They implemented more advanced integrity checks which I won't disclose here.

## HIDING PBOS IN DAYZ
DayZ and Arma 3 share little in common. DayZ even utilizes the new [Enfusion engine](https://enfusionengine.com/). However, a very apparent common denominator is poor security.
A prime example of this lackluster security is the lack of the updated addon security implemented in Arma's 1.86 update. That's right, DayZ never updated their addon security and is still vulnerable to the same PBO hider attacks that Arma 3 was. 

### IMPLEMENTATION
The implementation of such a PBO hider for DayZ is *identical* to an implementation for Arma 3 and is [extremely](https://github.com/Sherman0236/PBO-Ninja/blob/main/PBO-Ninja/GUI.cpp#L75) [simplistic](https://github.com/Sherman0236/PBO-Ninja/blob/main/PBO-Ninja/SDK/Engine.cpp#L36). 

1. Locate the GFileBanks .data pointer
* This can be done through basic static analysis. Open DayZ_x64.exe in your favorite static analysis tool such as IDA Pro or Ghidra and search for the following string: `*** ALL SIGNATURES CHECK START ***`
* Look for the only function that references the string. Just below the utilization of the string, there is a check ensuring the size of GFileBanks is greater than zero.
* Just further down in the disassembly, there is also a loop that iterates over every QFBank within GFileBanks. The first operation the loop performs is getting the current index inside GFileBanks.
* Success! We now know where the GFileBanks is located relative to the base of the image. 
![String Usage No Vars](/assets/img/posts/dz-pbo/string-usage-no-vars.png)
![String Usage](/assets/img/posts/dz-pbo/string-usage.png)

2. Find the index of the desired QFBank
* We need some basic information about the memory structure of a QFBank so we can identify the specific QFBank we want to delete. My [implementation of the QFBank class](https://github.com/Sherman0236/PBO-Ninja/blob/main/PBO-Ninja/SDK/QFBank.hpp) provides an easy-to-use class for interacting with QFBanks.
* Next, identifying information about the PBO, such as the name, must be gathered from the user. This could be done in any number of ways, such as a [command line](https://www.unknowncheats.me/forum/arma-3/140199-PBOhider.html) or [GUI](https://github.com/Sherman0236/PBO-Ninja/blob/main/PBO-Ninja/GUI.cpp#L75). 
* Iterating over each index of GFileBanks, we can compare certain fields of each QFBank against the user input. Once the identifying information is found, we know we have reached the index of the desired QFBank. 

3. Remove the QFBank
* To recap, we have the following information so far: The location of GFileBanks in memory, the size of GFileBanks, and the index of the desired QFBank. Now let's use all of that information!
* The integrity of GFileBanks must be preserved when removing a QFBank. To achieve this, we need to shift all subsequent QFBanks one position to the left. Otherwise, the array will not be contiguous, and the game will crash. 
* Once the QFBank has been removed, the size of GFileBanks should be adjusted accordingly. This is easily accomplished by decrementing the size value of GFileBanks by one.
* We have now removed a QFBank from GFileBanks and **will not be kicked** when joining secure multiplayer environments with additional addons!

## CONCLUSION
Although this technique can be used in malicious ways, this post is meant for educational purposes only. It's essential to understand the game's security mechanisms to create more secure and enjoyable gaming experiences.
Please use this information responsibly and ethically. 

#### Disclaimer
Any information in this post could become outdated at any point in time. The content of this blog post is intended for educational purposes only. By reading this blog post, you agree to use the information solely for educational purposes and to abide by all applicable laws and regulations concerning the protection of intellectual property. The author and publisher of this blog post shall not be held responsible for any damages, direct or indirect, arising from the misuse of the information provided.