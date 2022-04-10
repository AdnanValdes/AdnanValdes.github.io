+++
uid = 202203071616
title = "Zork Server"
date = 2022-04-10
[taxonomies]
tags = ["servers", "ssh", "terminal"]
+++

I love text adventure games. I am not very good at them, but I very much enjoy the world building and interactive aspect of text-based terminal games, in much the same way that I love tabletop RPGs like *Dungeons and Dragons* or *Stars Without Number*.

Text adventures are also excellent candidates for in-flight entertainment on long-haul routes. I recently travelled to Delhi from Vancouver, on a flight that was scheduled 3 days after the beginning of the Russian invasion of Ukraine. Because of the closure of Russian airspace to most airlines, my route was YVR -> YYZ (3 hour stop) YYZ -> Delhi. In other words, the ETE changed from about 13 hours to just over 22 hours.
<!-- more -->
To set up, I bought the Zork Anthology through Steam. I know Infocom has made Zork 1 - 3 [available for free](http://www.infocom-if.org/downloads/downloads.html), but because this was a fairly late schedule change I decided to go the easy way and shell out $8.00 for the whole set.

The games themselves work well, but I had a few problems with the set-up. To start with, I have Manjaro installed on all my computers, but the Zork Anthology only has a Windows release on Steam. Not much of an issue in itself: I regularly use Proton to play on Linux and I have had very few problems. However, the thought of running Proton to *then* run a DOS emulator to play Zork rubbed me the wrong way.

The other option I've seen is to play Zork through a web-based console, such as the one on [text adventures](http://textadventures.co.uk/games/play/5zyoqrsugeopel3ffhz_vq). It's a good way of making the game available to a more general audiene. What it lacks, I think, is correct aesthetic. That is, when running the game on a terminal all you see is the game's text and your prompt - what's more, you can have the terminal use your personalized colors and font. The browser version, on the other hand, has a much smaller play area, uses a single console theme, and is full of distractions, with tabs, links, ads, JS scripts, etc.

After landing in Delhi, then, I decided I would create my own Zork server. Originally I wanted to run it in my home, using Proxmox and LXC containers, but because I also wanted to make it available to the public, the end result is a GCP instance.


# Setting up an SSH Infocom server

The first decision I made was to use SSH for access into the server. I looked around and saw that there were a few Telnet servers that others have created in the past few years. You could argue that Telnet is a more appropriate protocol for a retro game (which allows for easier access, given the nature of Telnet itself). However, on a Windows machine the Telnet client has to be explicitly enabled via the command line or through the system settings interface; this alone adds an extra barrier to entry for my friends who are not huge computer nerds.

Additionally, SSH is a more secure protocol. I don't think that transmitting commands for Zork or other text-adventure games allows for the exploitation of particularly sensitive data. Yet, given that OpenSSH is available by default on most systems, choosing the more secure option has the benefit of creating less friction.

## Game files

While I could have downloaded the Zork games from the aforementioned Infocom website, I figured the Zork Anthology included the files themselves - files probably too old to have any sort of DRM. Sure enough, navigating to the anthology's local game files I found several DOSBOX `exe` files as well as a `.dat` file for Zork 1 through 3, Beyond Zork, and Planetfall.

With those in hand, I had to find a program to run a [Z-machine](https://www.wikiwand.com/en/Z-machine). Fortunately, this was as easy as installing [Frotz](https://davidgriffith.gitlab.io/frotz/):

```bash
sudo apt install frotz
```

Then, all I had to do was run `frotz zork.dat` and I was well on my way to being eaten by a grue.

## Setting up an SSH server

At this point I had the games running locally, but that's not very useful if I'm somewhere else or if I want my friends to access the game. I decided to run the server on the free tier of some cloud provider, mostly to avoid opening port 22 on my home router and because I didn't want strangers SSH'ing into my computers (see below for security!). The other advantage of using a cloud provider was that most of the time I would have access to a static IP, so I would not have to worry about dynamic DNS or using an API to monitor any changes that my ISP makes.

After shopping around for a bit, I settled on using Google's Compute Engine. This is partly out of familiarity - I already had a Pi-Hole instance running there - but mostly because they offer a perpetual free tier (unlike AWS, which is a 12 month trial).

Setting up the VM was a breeze, too. The free tier allows you to use an E2-micro general purpose VM with 10GB of storage space, 2 vCPUs and 1GB of memory. Far more than enough for 6 games from the late 70s/early 80s that are maybe 1 MB each. 

I don't think it matters much, but I chose Debian 11 as my gues OS. I originally tried all of this with Alpine, but that was far too much work and I couldn't make it work.

### Creating root access for set up

Once the VM was running, I used the Google Console to edit the instance's `sshd_config` file. Specifically, I allowed root access with a password (this was mostly out of frustration with trying to set up SSH keys from my computer). This was intended to be a temporary measure, just to allow me to install all the required components using my own terminal, instead of the browser-based version you get from the Google dashboard.

```
PermitRootLogin yes
StrictMode yes
MasAuthTries 3
MassSessions 1

PasswordAuthentication yes
PermitEmptyPasswords no
```
### Installing frotz and firejail

Once in the VM, I installed `frotz` and `firejail`:

```bash
sudo apt install frotz firejail
```
As I mentioned above, `frotz` is the interpreter used to run the games themselves. `firejail`, on the other hand, was installed as an added level of security. I am no pen tester by any means, but the firejail [website](https://firejail.wordpress.com/) claims that it "reduces the risk of security breaches by restricting the running environment of untrusted applications using Linux namespaces and seccomp-bpf".

From what I gather, this translates (at the risk of being rather reductive) to creating a sandbox or "container" wherein the relevant application is running. As a result, the program itself is less likely to be exploited to gain access to other parts of the system.

The main thing about it was that the promise of a sandbox environment coupled with ridiculous ease of use (in my case at least) means there is very little risk to me. To use firejail along with frotz, this is the command I had to run (note: I created a different user to try this, since firejail sensibly refused to be run as root):

```bash
firejail frotz zork.dat
```

And just like that, the game started up. There is still some `stdout` stuff on the terminal that I don't know how to remove, but at least it suggests something is working.

### adding new user, moving .DAT files

The next step was to create a specific user that would interact with the games - that is, a user that everyone has access to. Because this user is the public facing side of the set up, I also wanted to ensure that it was fairly restricted. The solution is to use `rbash`, a restrictive version of `bash` that still allows the user to use the terminal with appropriate restrictions.

I started by changing the user's shell and adding the user's bin directory:

```bash
chsh -s /bin/rbash zork
touch /home/zork/.bashrc
mkdir -p /home/zork/user/bin
echo "export PATH=/home/zork/usr/bin" >> .bashrc
```

At this point, the `zork` user is unable to run any commands, since there are no programs to run in their personal `bin`. Given the nature of this set up, the only binaries that should exist in this `bin` directory are `firejail` and `frotz`:

```bash
ln -s /usr/bin/firejail /home/zork/usr/bin/firejail
ln -s /usr/bin/frotz /home/zork/usr/bin/frotz
```

Note that I also had to set the right permissions in order for this user to access the new directory:

```bash
chmod -R 750 /home/zork
chown -R zork:zork /home/zork
```


Finally, I used `scp` to move the .DAT files that came with the anthology into the server. I renamed the files and removed the extension for ease of use, and so that I could add the following lines in the zork user's .bashrc:

```bash
alias zork="firejail frotz zork"
alias zork2="firejail frotz zork2"
alias zork3="firejail frotz zork3"
alias beyond_zork="firejail frotz beyond_zork"
alias planetfall="firejail frotz planetfall"
```

With this, anybody that connects to the server can only call `firejail` or `frotz`, and with the aliases it becomes quite easy to start any of the games.

### adding instructions
I added this to the server's `motd` file:

```
######################
WELCOME TO PLANETFALL!
######################

This is a microserver designed entirely for Infocom text adventures
To play, simply type the name of the game you want to play
Available games are:
- zork
- zork2
- zork3
- beyond_zork
- planetfall

Have fun, and try not to get eaten by a grue.
```

I figured that is enough information to at least get started. I might change that in the future. I also still want to figure out a way to call it when a user types `help`, for example.

### securing all files
To ensure that none of the files would be changed (no additions to the `bin` directory, for example), I used `chattr` (short for "change attributes").

Conveniently, one of the attributes that one can set is `i` to make it immutable. From the `chattr` man page:

```
A file with the 'i' attribute cannot be modified: it cannot be deleted or renamed, no link can  be created  to  this  file,  most of the file's metadata can not be modified, and the file can not be opened in write mode.  Only the superuser or a process possessing the CAP_LINUX_IMMUTABLE capability can set or clear this attribute.
```

Thus, `chattr +i /home/zork`.
