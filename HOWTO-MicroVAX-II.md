How to netboot a MicroVAX II from a Linux boot server
=====================================================

<toby@telegraphics.com.au>
October, 2012.

The NetBSD distribution used was 1.4.1.


### MicroVAX II configuration

    7606 AF		KA630 MicroVAX II CPU (incl 1MB)
    7609 AH		MS630 8MB parity RAM
    9047		grant continuity
    7516YM		DELQA Ethernet (MAC 08002B13F87D)
    3104		DHV11
    7164	\	KDA50 1/2
    7165	/	KDA50 2/2
    7555		RQDX3
    7546		TUK50


### My network

For reference, I have used these addresses on my LAN.

    10.0.0.3    pc       # linux server
    10.0.0.5    gateway
    10.0.0.63   mvii     # the client MicroVAX II


### Enable multicast in linux kernel

    # grep MULTIC /usr/src/linux/.config
    CONFIG_IP_MULTICAST=y


### Enable multicast on Ethernet

    # ifconfig eth0 allmulti
    # ifconfig
    eth0      Link encap:Ethernet  HWaddr 00:30:1b:b5:5d:49
              inet addr:10.0.0.3  Bcast:10.0.0.255  Mask:255.255.255.0
              UP BROADCAST RUNNING ALLMULTI MULTICAST  MTU:1500  Metric:1
              RX packets:1459 errors:0 dropped:0 overruns:0 frame:0
              TX packets:1149 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:1000
              RX bytes:136386 (133.1 KiB)  TX bytes:157410 (153.7 KiB)
              Interrupt:19


### Check out and build mopd

From [this repository](https://github.com/qu1j0t3/mopd/).


### tftpd must be running

    # emerge tftp-hpa
    # cat /etc/conf.d/in.tftpd
        # Path to server files from
        # Depending on your application you may have to change this.
        # This is commented out to force you to look at the file!
        #INTFTPD_PATH="/var/tftp/"
        INTFTPD_PATH="/tftpboot/mop"
        #INTFTPD_PATH="/tftproot/"

        # For more options, see in.tftpd(8)
        # -R 4096:32767 solves problems with ARC firmware, and obsoletes
        # the /proc/sys/net/ipv4/ip_local_port_range hack.
        # -s causes $INTFTPD_PATH to be the root of the TFTP tree.
        # -l is passed by the init script in addition to these options.
        INTFTPD_OPTS="-l -R 4096:32767 -s ${INTFTPD_PATH}"

Note that these must not be symlinks (or tftpd won't serve them):

    # mkdir -p /tftpboot/mop/
    # cd /tftpboot/mop/
    # cp ~toby/NetBSD-1.4.1/vax/installation/netboot/boot.mop MOPBOOT.SYS
    # cp ~toby/NetBSD-1.4.1/vax/installation/netboot/boot.mop 08002b13f87d.SYS

    # ls -lR /tftpboot/
    /tftpboot/:
    total 0
    drwxr-xr-x 2 root root 112 Oct  5 21:41 mop

    /tftpboot/mop:
    total 144
    -rw-r--r-- 2 root root 72192 Oct  5 21:40 08002b13f87d.SYS
    -rw-r--r-- 2 root root 72192 Oct  5 21:40 MOPBOOT.SYS

    # /etc/init.d/in.tftpd start


### nfs server must be running

Follow the diskless setup instructions here:
http://www.netbsd.org/docs/network/netboot/intro.vax.html

    # cd /export/client/root
    # tar --numeric-owner -xpzf ~toby/NetBSD-1.4.1/vax/binary/sets/kern.tgz
    # mknod /export/client/root/dev/console c 0 0

    # cat /etc/exports
        /export/client/root mvii(rw,no_root_squash,no_subtree_check)
        /export/client/swap mvii(rw,no_root_squash,no_subtree_check)
        /export/client/usr mvii(rw,root_squash,no_subtree_check)
        /export/client/home mvii(rw,root_squash,no_subtree_check)

    # cat /export/client/root/etc/fstab
        10.0.0.3:/export/client/swap   none  swap  sw,nfsmntpt=/swap
        10.0.0.3:/export/client/root   /     nfs   rw 0 0
        10.0.0.3:/export/client/usr    /usr  nfs   rw 0 0
        10.0.0.3:/export/client/home   /home nfs   rw 0 0

    # diff /export/client/root/etc/rc.conf.orig /export/client/root/etc/rc.conf
    12c12
    < rc_configured=NO
    ---
    > rc_configured=YES
    20c20
    < hostname=""				# if blank, use /etc/myname
    ---
    > hostname="mvii"				# if blank, use /etc/myname
    24c24,25
    < defaultroute=""				# if blank, use /etc/mygate
    ---
    > ifconfig_qe0="inet 10.0.0.63 netmask 255.255.255.0"
    > defaultroute="10.0.0.5"				# if blank, use /etc/mygate
    52c53
    < auto_ifconfig=YES				# config all avail. interfaces
    ---
    > auto_ifconfig=NO				# config all avail. interfaces
    97c98
    < nfs_client=NO					# enable client daemons
    ---
    > nfs_client=YES					# enable client daemons
    118a120
    >

Filesystems after installing some sets:

    # du -sh /export/client/*
    41M	/export/client/home
    9.7M	/export/client/root
    17M	/export/client/swap
    0	/export/client/sys
    52M	/export/client/usr
    152K	/export/client/var


### mopd must be running

Successful boot log shown.

    # mopd/mopd -d eth0
    mopd: not running as daemon, -d given.
    MOP DL 8:0:2b:13:f8:7d   > ab:0:0:1:0:0      len   11 code 08 RPR
    MOP DL 0:30:1b:b5:5d:49  > 8:0:2b:13:f8:7d   len    1 code 03 ASV
    MOP DL 8:0:2b:13:f8:7d   > 0:30:1b:b5:5d:49  len   11 code 08 RPR
    Native Image (VAX)
    Header Block Count: 1
    Image Size:         00011800
    Load Address:       00000000
    Transfer Address:   00000000
    MOP DL 0:30:1b:b5:5d:49  > 8:0:2b:13:f8:7d   len 1006 code 02 MLD
    MOP DL 8:0:2b:13:f8:7d   > 0:30:1b:b5:5d:49  len    3 code 0a RML
    ...
    MOP DL 0:30:1b:b5:5d:49  > 8:0:2b:13:f8:7d   len  686 code 02 MLD
    MOP DL 8:0:2b:13:f8:7d   > 0:30:1b:b5:5d:49  len    3 code 0a RML
    MOP DL 0:30:1b:b5:5d:49  > 8:0:2b:13:f8:7d   len   32 code 14 PLT
    MOP DL 8:0:2b:13:f8:7d   > 0:30:1b:b5:5d:49  len    3 code 0a RML

And the following is logged in syslog:

    Oct  6 20:25:53 localhost mopd[8125]: 8:0:2b:13:f8:7d (1) Do you have 08002b13f87d? (Yes)
    Oct  6 20:25:53 localhost mopd[8125]: 8:0:2b:13:f8:7d Send me 08002b13f87d
    Oct  6 20:25:53 localhost mopd[8125]: hostname: [ipc] len: 3
    Oct  6 20:25:53 localhost mopd[8125]: 8:0:2b:13:f8:7d Load completed


### bootpd must be running

    # emerge -av netkit-bootpd

(Note, /tftpboot/boot.netbsd doesn't actually exist, and isn't part of
NetBSD. This setting seems to be ignored?)

    # cat /etc/bootptab
    mvii:\
            :ht=ether:\
            :ha=08002B13F87D:\
            :ip=10.0.0.63:\
            :bf=/tftpboot/boot.netbsd:\
            :rp=/export/client/root/:

I was not able to get this to work via xinetd, so I started the daemon
standalone, manually.

Successful boot log shown.

    # bootpd -d 9 -s
    bootpd: info(6):   bootptab mtime: Sat Oct  6 19:31:29 2012
    bootpd: info(6):   reading "/etc/bootptab"
    bootpd: info(6):   read 1 entries (1 hosts) from "/etc/bootptab"
    bootpd: info(6):   recvd pkt from IP addr 0.0.0.0
    bootpd: info(6):   bootptab mtime: Sat Oct  6 19:31:29 2012
    bootpd: info(6):   request from Ethernet address 08:00:2B:13:F8:7D
    bootpd: info(6):   found 10.0.0.63 (mvii)
    bootpd: info(6):   bootfile="/tftpboot/boot.netbsd"
    bootpd: info(6):   vendor magic field is 99.130.83.99
    bootpd: info(6):   request message length=548
    bootpd: info(6):   extended reply, length=548, options=312
    bootpd: info(6):   sending reply (with RFC1048 options)
    bootpd: info(6):   setarp 10.0.0.63 - 08:00:2B:13:F8:7D
    ..pause..
    bootpd: info(6):   recvd pkt from IP addr 0.0.0.0
    bootpd: info(6):   bootptab mtime: Sat Oct  6 19:31:29 2012
    bootpd: info(6):   request from Ethernet address 08:00:2B:13:F8:7D
    bootpd: info(6):   found 10.0.0.63 (mvii)
    bootpd: info(6):   bootfile="/tftpboot/boot.netbsd"
    bootpd: info(6):   vendor magic field is 99.130.83.99
    bootpd: info(6):   request message length=548
    bootpd: info(6):   extended reply, length=548, options=312
    bootpd: info(6):   sending reply (with RFC1048 options)
    bootpd: info(6):   setarp 10.0.0.63 - 08:00:2B:13:F8:7D


### Serial console on MicroVAX II

    KA630-A.V1.3

    Performing normal system tests.

      7..6..5..4..3..

    Tests completed.


    >>> b xqa0

      2..1..0..


    >> NetBSD/vax boot [Aug 13 1999 20:18:28] <<
    >> Press any key to abort autoboot
    > boot netbsd
    Trying BOOTP
    Using IP address: 10.0.0.63
    myip:  (10.0.0.63), mask: 255.0.0.0
    root addr=10.0.0.3 path=/export/client/root/
    841724+43856+159808+[64440+72748] total=0x120b78
    Copyright (c) 1996, 1997, 1998, 1999
        The NetBSD Foundation, Inc.  All rights reserved.
    Copyright (c) 1982, 1986, 1989, 1991, 1993
        The Regents of the University of California.  All rights reserved.

    NetBSD 1.4.1 (GENERIC) #13: Fri Aug 13 05:35:08 PDT 1999
        root@futplex:/usr/src/sys/arch/vax/compile/GENERIC

    MicroVAX II
    realmem = 9428992
    avail mem = 6565888
    Using 115 buffers containing 471040 bytes of memory.
    mainbus0 (root)
    cpu0 at mainbus0: KA630
    uba0 at mainbus0: Q22
    mtc0 at uba0 csr 174500 vec 774 ipl 17
    mscpbus0 at mtc0: version 4 model 3
    mscpbus0: DMA burst size set to 4
    mt0 at mscpbus0 drive 0: TK50
    uda0 at uba0 csr 172150 vec 770 ipl 17
    mscpbus1 at uda0: version 4 model 3
    mscpbus1: DMA burst size set to 4
    uda1 at uba0 csr 160334 vec 764 ipl 17
    mscpbus2 at uda1: version 8 model 13
    mscpbus2: DMA burst size set to 4
    qe0 at uba0 csr 174440 vec 760 ipl 17
    qe0: delqa, hardware address 08:00:2b:13:f8:7d
    boot device: qe0
    nfs_boot: trying DHCP/BOOTP
    nfs_boot: BOOTP server: 0xa000003
    nfs_boot: my_addr=0xa00003f
    nfs_boot: my_mask=0xff000000
    root on deepthought:/export/client/root/
    Clock has lost 15609 day(s) - CHECK AND RESET THE DATE.
    root file system type: nfs
    Automatic boot in progress: starting file system checks.
    setting tty flags
    starting network
    hostname: mvii
    configuring network interfaces:.
    add net default: gateway 10.0.0.5
    swapctl: adding 10.0.0.3:/export/client/swap as swap device at priority 0
    starting system logger
    checking for core dump...
    savecore: no core dump (no dumpdev)
    starting rpc daemons: portmap.
    starting nfs daemons: nfsiod.
    creating runtime link editor directory cache.
    ldconfig: /usr/pkg/lib: No such file or directory
    ldconfig: /usr/X11R6/lib: No such file or directory
    setting securelevel: kern.securelevel: 0 -> 1
    checking quotas: done.
    building databases...
    clearing /tmp
    updating motd.
    standard daemons: update cron.
    starting network daemons: inetd.
    starting local daemons:.
    Sat Jan 10 12:19:41 PST 1970

    NetBSD/vax (mvii) (console)

    login:


### Troubleshooting

    $ telnet mvii
    Trying 10.0.0.63...
    Connected to mvii.
    Escape character is '^]'.
    telnetd: All network ports in use.
    Connection closed by foreign host.

This occurs if you did not run `MAKEDEV all` earlier, per instructions:
http://www.netbsd.org/docs/network/netboot/files.html

Fix:

    mvii# cd /dev
    mvii# ./MAKEDEV pty0
    or
    mvii# ./MAKEDEV all

### Other information

 * http://lakesdev.blogspot.co.uk/2009/04/netbsd-bbs-and-vaxen.html
 * NetBSD installation: http://mirror.planetunix.net/pub/NetBSD/NetBSD-archive/NetBSD-1.4.1/vax/INSTALL.html