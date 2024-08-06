# youtubeUnblock
Bypasses Googlevideo detection systems that relies on SNI. The package is for Linux only. The package is fully compatible with routers running OpenWRT. To learn how to build the package on OpenWRT, consult [this chapter](https://github.com/Waujito/youtubeUnblock?tab=readme-ov-file#openwrt-case).

For Windows use [GoodbyeDPI from ValdikSS](https://github.com/ValdikSS/GoodbyeDPI) (you can find how to use it for YouTube [here](https://github.com/ValdikSS/GoodbyeDPI/issues/378)) The same behavior is also implemented in [zapret package for linux](https://github.com/bol-van/zapret).

## How it works:
Lets look from the DPIses side of view: All they have is ip and tcp information, higher-level data is encrypted. So from the IP header only IP address might be helpful for them. In tcp here is basically nothing. So they may handle IP addresses and process it. What's wrong? Google servers are on the way: It is very hard to handle all that infrastracture. One server may host multiple websites and it is very bad if them block, say Google Search trying to block googlevideo. But even if googlevideo servers have their own ip for only googlevideo purposes, here is a problem about how large is Google infrastracture and how much servers are here. The DPIs can't even parse normally all the servers, because each video may live on it's cache server. So what's else? Let's take a look at a TLS level. All information here is encrypted. All... Except hello messages! They are used to initialize handshake connections and hold tons of helpful information. If we talk about TLS v1.3, it is optimized to transfer as less information as possible unencrypted. But here is only one thing that may point us which domain the user wants to connect, the SNI extension. It transfers all domain names unencrypted. Exactly what we need! And DPIs may use this thing to detect google video connections and slow down them (In fact they are corrupting a tcp connection with bad packets).

So we aims to somehow hide the SNI from them. How?
- We can alter the SNI name in the tls packet to something else. But what's wrong with this? The server also uses SNI name for certificates. And if we change it, the server will return an invalid certificate which browser can't normally process, which may turn out to the MITM problem.
- We can encrypt it. Here are a lot of investigations about SNI, but the server should support the technique. Also ISPs may block encrypted SNI. [Check this Wikipedia page](https://en.wikipedia.org/wiki/Server_Name_Indication)
- So what else can we do with the SNI info? If we can't hide it, let's rely on DPIs weak spots. The DPI is an extremly high loaded machine that analyzes every single packet sent to the Internet. And every performance-impacted feature should be avoided for them. One of this features is IP packet fragmentation. We can split the packet in the middle of SNI message and post it. For DPI fragmentation involves too much overhead: they should store a very big mapping table which maps IP id, Source ip and Destination ip. Also note that some packets may be lost and DPI should support auto-clean of that table. So just imagine how much memory and CPU time will this cost for DPI. But fragments are ok for clients and hosts. And that's the base idea behind this package. I have to mention here that the idea isn't mine, I get in here after some research for this side. Here already was a solution for Windows, GoodbyeDPI. I just made an alternative for Linux.

You may read further in an [yt-dlp issue page](https://github.com/yt-dlp/yt-dlp/issues/10443) and in [ntc party forum](https://ntc.party/t/%D0%BE%D0%B1%D1%81%D1%83%D0%B6%D0%B4%D0%B5%D0%BD%D0%B8%D0%B5-%D0%B7%D0%B0%D0%BC%D0%B5%D0%B4%D0%BB%D0%B5%D0%BD%D0%B8%D0%B5-youtube-%D0%B2-%D1%80%D0%BE%D1%81%D1%81%D0%B8%D0%B8/8074).

## How it processes packets
When the packet is joining the queue, the application checks sni payload to be googlevideo (right how the DPIs do), segmentates/fragmentates (both TCP and IP fragmentation techniques are supported) and posts the packet. Note that it is impossible to post two fragmented packets from one netfilter queue verdict. Instead, the application drops an original packet and makes another linux raw socket to post the packets in the network. To escape infinity loops the socket marks outgoing packets and the application automatically accepts it.

## Usage:
Before compilation make sure `gcc`, `make`, `autoconf`, `automake`, `pkg-config` and `libtool` is installed. For Fedora `glibc-static` should be installed as well.
Compile with `make`. Install with `make install`. The package include libnetfilter_queue, libnfnetlink and libmnl as static dependencies. The package requires linux-headers and kernel built with netfilter nfqueue support.

You should also configure iptables for this to start working:
```iptables -A OUTPUT -p tcp --dport 443 -j NFQUEUE --queue-num 537 --queue-bypass``` (or do ```nft add rule ip mangle OUTPUT tcp dport 443 counter queue num 537 bypass``` for nftables.)
Here iptables serves every tcp packet, destinating port 443 for this userspace packet analyzer (via netfilter kernel module) queue-num may be any number from 0 to 65565. --queue-bypass allows traffic to pass if the application is down.

Also tips to explicitly accept all important outgoing raw packets from youtubeUnblock from [Troubleshooting EPERMS](https://github.com/Waujito/youtubeUnblock?tab=readme-ov-file#troubleshooting-eperms-operation-not-permitted) may be useful to avoid issues.

Run an application with `youtubeUnblock 537` where `537` stands for the queue-num (must be the same as in the iptables rule).

Systemd daemon is also available. Do `systemctl enable --now youtubeUnblock.service` after installation (uses queue-num `537`). Please, note that systemd will configure iptables automatically. If you have troubles with it, delete ExecStartPre and ExecStop from youtubeUnblock.service and configure iptables manually (may be a useful case for nftables).

If you don't want youtubeUnblock to log on each googlevideo request pass -DSILENT as CFLAGS.

Also DNS over HTTPS (DOH) is preferred for additional anonimity. 

## Troubleshooting
If you have any troubles with youtubeUnblock, here are some options to tune. If them don't work in your case, please, open an issue. You can pass these options in make CFLAGS (`make CFLAGS=...`) or edit CFLAGS variable in Makefile.
Available flags:
- -DUSE_SEG2_DELAY This flag forces youtubeUnblock to wait little bit before send the 2nd part of the split packet. You can tune the amount of time in `#define SEG2_DELAY 100` where 100 stands for milliseconds.
- -DNO_FAKE_SNI This flag disables -DFAKE_SNI which forces youtubeUnblock to send at least three packets instead of one with TLS ClientHello: Fake ClientHello, 1st part of original ClientHello, 2nd part of original ClientHello. This flag may be related to some Operation not permitted error messages, so befor open an issue refer to FAQ for EPERMS.
- -DNOUSE_GSO This flag disables fix for Google Chrome fat ClientHello. The GSO is well tested now, so this flag probably won't fix anything.

If you are on Chromium you may have to disable kyber (the feature that makes the TLS ClientHello very fat). I've got the problem with it on router, so to escape possibly errors it is better to just disable it: in chrome://flags search for kyber and switch it to disabled state. 

### Troubleshooting EPERMS (Operation not permitted)
EPERM may occur in a lot of places but generally here are two: mnl_cb_run and when sending the packet via rawsocket (raw_frags_send and send fake sni).
- mnl_cb_run Operation not permitted indicates that another instance of youtubeUnblock is running on the specified queue-num.
- rawsocket Operation not permitted indicates that the packet is being dropped by nefilter rules. In fact this is a hint from the kernel that something wrong is going on and we should check the firewall rules. Before dive into the problem let's make it clean how the mangled packets are being sent. Nefilter queue provides us with the ability to mangle the packet on fly but that is not suitable for this program because we need to split the packet to at least two independent packets. So we are using [linux raw sockets](https://man7.org/linux/man-pages/man7/raw.7.html) which allows us to send any ipv4 packet. **The packet goes from the OUTPUT chain even when NFQUEUE is set up on FORWARD (suitable for OpenWRT).** So we need to escape packet rejects here.
    * raw_frags_send EPERM: just make sure outgoing traffic is allowed (RELATED,ESTABLISHED should work, if not, go to step 3)
    * send fake sni EPERM: Fake SNI is out-of-state thing and will likely corrupt the connection (the behavior is expected). conntrack considers it as an invalid packet. By default OpenWRT set up to drop outgoing packets like this one. You may delete nftables/iptables rule that drops packets with invalid conntrack state, but I don't recommend to do this. The step 3 is better solution.
    * Step 3, ultimate solution. Use mark (don't confuse with connmark). The youtubeUnblock uses mark internally to avoid infinity packet loops (when the packet is sent by youtubeUnblock but on next step handled by itself). Currently it uses mark (1 << 15) = 32768. You should put iptables/nftables that ultimately accepts such marks at the very start of the filter OUTPUT chain: `iptables -I OUTPUT -m mark --mark 32768/32768 -j ACCEPT` or `nft insert rule inet fw4 output mark and 0x8000 == 0x8000 counter accept`.

## Performance 
If you have bad performance you can queue to youtubeUnblock only first, say, 20 packets from the connection. To do so, use nftables conntrack packets counter: `nft add rule inet fw4 mangle_forward tcp dport 443 ct original "packets < 20" counter queue num 537 bypass`. For my 1 CPU core device it worked pretty well. This works because we do care about only first packets with ClientHello. We don't need to process others.

The same behavior is also possible in iptables: `iptables -t mangle -A FORWARD -p tcp -m tcp --dport 443 -m connbytes --connbytes-dir original --connbytes-mode packets --connbytes 0:19 -j NFQUEUE --queue-num 537 --queue-bypass`. (The package iptables-mod-conntrack-extra is required for connbytes on OpenWRT)

For hosts change FORWARD to OUTPUT.


## OpenWRT case
The package is also compatible with routers. The router should be running by linux-based system such as [OpenWRT](https://openwrt.org/).
You can build under openwrt with two options: first - through the SDK, which is preferred way and second is cross-compile manually with openwrt toolchain.

### Building OpenWRT .ipk package
OpenWRT provides a high-level SDK for the package builds. 
First step is to download or compile OpenWRT SDK for your specific platform. The SDK can be compiled according to [this tutorial](https://openwrt.org/docs/guide-developer/toolchain/using_the_sdk). Beside of raw source code of SDK, OpenWRT also offers precompiled SDKs for your router. You can find it on the router page. For example, I have ramips/mt76x8 based router so for me the sdk is on https://downloads.openwrt.org/releases/23.05.3/targets/ramips/mt76x8/ and called `openwrt-sdk-23.05.3-ramips-mt76x8_gcc-12.3.0_musl.Linux-x86_64`. You will need to [install sdk requirements on your system](https://openwrt.org/docs/guide-developer/toolchain/install-buildsystem) If you have any problems, use docker ubuntu:24.04 image. Make sure to be a non-root user since some makesystem fails with it. Next, untar the SDK and cd into it. Do `echo "src-git youtubeUnblock https://github.com/Waujito/youtubeUnblock.git;openwrt" >> feeds.conf`, `./scripts/feeds update youtubeUnblock`, `./scripts/feeds install -a -p youtubeUnblock`, `make package/youtubeUnblock/compile`. Now the packet is built and you can import it to the router. Find it in `bin/packages/<target>/youtubeUnblock/youtubeUnblock-<version>.ipk`. Go to your router interface and put it in via System-Software-install_package. Now the package is on the router. Goto System-Startup, restart firewall and start youtubeUnblock. You are done!

### Building with toolchain
The precompiled toolchain located near the SDK. For me it is called `openwrt-toolchain-23.05.3-ramips-mt76x8_gcc-12.3.0_musl.Linux-x86_64.tar.xz`. When you download the toolchain, untar it somewhere. Now we are ready for compilation. My cross gcc asked me to create a staging dir for it and pass it as an environment variable. Also you should notice toolsuite packages and replace my make command with yours. ```STAGING_DIR=temp make CC=/usr/bin/mipsel-openwrt-linux-gcc LD=/usr/bin/mipsel-openwrt-linux-ld AR=/usr/bin/mipsel-openwrt-linux-ar OBJDUMP=/usr/bin/mipsel-openwrt-linux-objdump NM=/usr/bin/mipsel-openwrt-linux-nm STRIP=/usr/bin/mipsel-openwrt-linux-strip CROSS_COMPILE_PLATFORM=mipsel-buildroot-linux-gnu```. Take a look at `CROSS_COMPILE_PLATFORM` It is required by autotools but I think it is not necessary. Anyways I put `mipsel-buildroot-linux-gnu` in here. For your model may be an [automake cross-compile manual](https://www.gnu.org/software/automake/manual/html_node/Cross_002dCompilation.html) will be helpful. When compilation is done, the binary file will be in build directory. Copy it to your router. Note that an ssh access is likely to be required to proceed. sshfs don't work on my model so I injected the application to the router via Software Upload Package page. It has given me an error, but also a `/tmp/upload.ipk` file which I copied in root directory, `chmod +x`-ed and run.

### Configuration
If you compiled the package via the SDK everything is preinstalled. But if you got any issues (suitable for routers with iptables instead of nftables), ssh into the router and check up everything manually.

For iptables: install a normal iptables user-space app: `xtables-legacy iptables-zz-legacy` and kernel/iptables nfqueue extensions: `iptables-mod-nfqueue kmod-ipt-nfqueue` and add `iptables -t mangle -A FORWARD -p tcp -m tcp --dport 443 -j NFQUEUE --queue-num 537 --queue-bypass` rule. 

If you prefer nftables, this should work: `nft add rule inet fw4 mangle_forward tcp dport 443 counter queue num 537 bypass`. Note that `kmod-nft-queue` should be installed.

Also you can copy `owrt/537-youtubeUnblock.nft` to `/usr/share/nftables.d/ruleset-post/537-youtubeUnblock.nft` and run `/etc/init.d/firewall reload`. This will reload the nftables ruleset and automatically link 537-youtubeUnblock.nft with it.

Next step is to daemonize the application in openwrt. Copy `owrt/youtubeUnblock.owrt` to `/etc/init.d/youtubeUnblock` and put the program into /usr/bin/. (Don't forget to `chmod +x` both). Now run `/etc/init.d/youtubeUnblock start`. You can alo run `/etc/init.d/youtubeUnblock enable` to force OpenWRT autostart the program on boot, but I don't recommend this since if the packet has bug you may lose access to the router (I think you will be able to reset it with reset settings tricks documented for your router). 

## If you have any questions/suggestions/problems feel free to open an issue.
