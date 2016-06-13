# SLAACer - Accountability with IPv6

SLAACer primary objective is to place a second option on the table for network sysadmin's when they deploy IPv6 in their organisation.  So far the mass consenus generally has bee that you must roll out DHCPv6 if you want to be able to map your IPv6 addresses back to MAC addresses; and do not wish to poll via SNMP/CLI all your routers.

SLAACer embraces that in the IPv6 world, unlike the IPv4 world, multiple IP assignment to a node should be encouraged if not actually be considered the norm:

 * [EUI-64](http://en.wikipedia.org/wiki/IPv6_address#Modified_EUI-64) - subnet + MAC address
 * [privacy/temporary/randomised](http://en.wikipedia.org/wiki/IPv6_address#Temporary_addresses)
 * static - manual assignment
 * [Mobile IPv6](http://en.wikipedia.org/wiki/Mobile_IP) - non-local
 * [link-local](http://en.wikipedia.org/wiki/IPv6_address#Link-local_addresses_and_zone_indices) - 'fe80::/64'

Much like NAT, I have the opinion that DHCP has become one of those 'latent' efforts that sysadmins have become use to deploying in the IPv4 world when in actual fact they do require some degree of effort (especially to make highly available) which can be completely engineered away in the IPv6 world.  Some might argue that DHCP can also be used to give TFTP, NTP, LDAP, etc servers in their payload, I personally feel that using [DNS SRV records](http://en.wikipedia.org/wiki/SRV_record) and multicast [SLP](http://en.wikipedia.org/wiki/Service_Location_Protocol) are far more appropriate methods.

Either way and whatever your view, SLAACer now permits network sysadmin's to *choose* to use either [DHCPv6](http://en.wikipedia.org/wiki/DHCPv6) or [SLAAC](http://en.wikipedia.org/wiki/SLAAC#Stateless_address_autoconfiguration_.28SLAAC.29) depending on their needs.  A situation with options should always be considered preferable over a sitation where you are forced to embrace one method/vendor.

**N.B.** SLAACer is not *the* solution but should tie us all over until something better comes along (lets hope vendors give us something like SNMP traps for changes to the IPv6 to MAC mapping events).

## Issues

Do contact me if you want to contribute patches, send feedback or just generally have a chat about SLAACer.  Currently on the roadmap are the following changes that I can think of:

 * improve Perl POD documentation for the daemon
 * improvements to filtering port-mirroring to mirror only the traffic we are interested in (including developments into SPF loopback approaches)
 * schemas for [SQLite](http://www.sqlite.org/), [MySQL](http://www.mysql.com/), [MS-SQL](http://www.microsoft.com/sqlserver/) and others
 * configuration examples for other syslog servers such as [rsyslog](http://www.rsyslog.com/)
 * record and process ICMPv6 RA packets to track down mis-configured hosts

## Related Links

 * [SLAACers - IPv6 Accountability without DHCPv6](http://webmedia.company.ja.net/content/documents/shared/networkshop120411/clouter_slaacersipv6withoutdhcpv6butwithaccountability.pdf) ([Networkshop 39](http://www.ja.net/services/events/networkshop-39.html)) - presentation

## Overview

In it's ugliest form, SLAACer requires that you port-mirror *every* workstation edge switch port in your network into it. This is actually not as bad as it sounds as to stop things getting out of control you can use filters on the port-mirror'ing to capture only the traffic SLAACer is interested in ([ICMPv6 NS/NA traffic](http://en.wikipedia.org/wiki/Neighbor_Discovery_Protocol)), ingress only traffic and a number of other tricks discussed later.

All this traffic is sent to a dedicated Ethernet interface that the 'slaacer' daemon listens on (using [libpcap](http://www.tcpdump.org/)).  'slaacer' digests the traffic and sends out syslog messages depending on what it sees.  A syslog server collects these messages (additionally you can process the syslog traffic from your IPv4 ISC DHCP server too) to write to a text file as well as parsing the contents and creating an SQL query calling an function that records the information in an easy to use format.

## Requirements

 * perl 5.10 with [Net::Pcap](http://search.cpan.org/perldoc?Net::Pcap) install
 * [syslog-ng](http://www.balabit.com/network-security/syslog-ng)
 * an SQL server
 * [PostgreSQL](http://www.postgresql.org/)
 * a network infrastructure that supports *remote* port-mirroring (Cisco call this '[RSPAN](http://www.cisco.com/en/US/docs/switches/lan/catalyst6500/ios/12.2SXF/native/configuration/guide/span.html#wp1037601)')

## Usage

Once deployed and you have all your data trickling into your SQL server, data is recorded as the column 'start' indicating when the IP address (either IPv4 or IPv6) was first seen and 'last' when it was last recorded((**warning:** especially in the case of DHCP leases, 'last' and when the host was last using that address is only accurate to the last timestamp a DHCP renew request, or IPv6 NA packet, was sent)).

You can now start to make queries such as "what addresses has the MAC address 00-17-42-87-ce-f7 used since the beginning of March 2011?":
    
    user@host:~$ echo "SELECT * FROM mac2addr WHERE mac = '00-17-42-87-ce-f7' AND last > '2011-03-01' ORDER BY last DESC" | psql --host=%SQL-SERVER% %DATABASE% %USERNAME%
    Password for user %USERNAME%:
      id   |        mac        |                 addr                 |         start          |          last
    -------+-------------------+--------------------------------------+------------------------+------------------------
      6609 | 00-17-42-87-ce-f7 | fe80::217:42ff:fe87:cef7             | 2011-02-23 10:06:23+00 | 2011-04-08 14:55:33+01
     31403 | 00-17-42-87-ce-f7 | 2001:630:1b:8000:c4b:c8fa:d360:5b3   | 2011-04-07 08:32:43+01 | 2011-04-08 14:44:06+01
     10185 | 00-17-42-87-ce-f7 | 10.150.72.38                         | 2011-03-07 08:23:02+00 | 2011-04-08 07:41:30+01
     30027 | 00-17-42-87-ce-f7 | 2001:630:1b:8000:c161:3cde:da18:9caa | 2011-04-04 08:28:22+01 | 2011-04-06 15:58:03+01
     29489 | 00-17-42-87-ce-f7 | 2001:630:1b:8000:51bc:41fe:62de:99da | 2011-04-01 08:23:21+01 | 2011-04-01 20:11:08+01
    (5 rows)

The most useful request is, in the case of abuse reports, to ask "who was using the IP address 2001:630:1b:8000:c4b:c8fa:d360:5b3 at 2011-04-07 22:00?":

    user@host:~$ echo "SELECT * FROM mac2addr WHERE addr = '2001:630:1b:8000:c4b:c8fa:d360:5b3' AND start `<= '2011-04-07 22:00' AND last >`= '2011-04-07 22:00'" | psql --host=%SQL-SERVER% %DATABASE% %USERNAME%
    Password for user ac56:
      id   |        mac        |                addr                |         start          |          last
    -------+-------------------+------------------------------------+------------------------+------------------------
     31403 | 00-17-42-87-ce-f7 | 2001:630:1b:8000:c4b:c8fa:d360:5b3 | 2011-04-07 08:32:43+01 | 2011-04-08 14:44:06+01
    (1 row)

All the IPv6 information ((the IPv4 DHCP entries come from the regular ISC DHCP syslog messages)) is obtained from the syslog messages that are generated by 'slaacer':
    
    Apr 11 08:25:26: 00-17-42-87-ce-f7 2001:630:1b:8000:7c1a:d241:3902:b982 [SOLICIT,dad]
    Apr 11 08:25:27: 00-17-42-87-ce-f7 2001:630:1b:8000:217:42ff:fe87:cef7 [SOLICIT,dad]
    Apr 11 08:26:12: 00-17-42-87-ce-f7 2001:630:1b:8000:7c1a:d241:3902:b982 [ADVERT,solicited,override]
    Apr 11 08:56:10: 00-17-42-87-ce-f7 fe80::217:42ff:fe87:cef7 [ADVERT,solicited]
    Apr 11 08:56:11: 00-17-42-87-ce-f7 2001:630:1b:8000:7c1a:d241:3902:b982 [ADVERT,solicited]
    Apr 11 08:57:10: 00-17-42-87-ce-f7 fe80::217:42ff:fe87:cef7 [ADVERT,solicited]

### SLAACer

'slaacer' supports a number of parameters when being called (in addition to '--help, -h' and '--man'):

 * **--interface, -i:** listen for IPv6 ICMP packets on a particular interface (//required//)
 * **--debug, -d:** enable verbose debugging
 * **--nofork, -n:** do not fork into background
 * **--record=path, -R path:** path to place recordings in (default: /tmp)
 * **--playback=filename, -P filename:** playback a former recording and exit (implies 'nofork' but does no syslog'ing)

Whilst running, 'slaacer' supports receiving and acting on the following signals (ie. 'pkill -USR1 -x slaacer'):

 * **SIGTERM/SIGINT:** cleanly shutdown
 * **SIGUSR1:** write out traffic being received by the daemon to a PCAP compatible file ('/path/specified/for/record/slaacer.%Y-%m-%dT%TZ') and when signaled when already recording, stop recording and close the file

## Download

The [git](http://git-scm.com/) tree for hopefully everything you need can be found at [https://github.com/jimdigriz/slaacer](https///github.com/jimdigriz/slaacer).

For those unfamiliar with git, if you have installed it, all you need to type is:

    user@host:~$ git clone git://github.com/jimdigriz/slaacer.git
    user@host:~$ cd slaacer
    user@host:~/slaacer$ ls -l
    total 16
    -rwxr-xr-x 1 user group 12882 Apr 10 15:09 slaacer
    drwxr-xr-x 2 user group    18 Apr 10 15:09 sql
    drwxr-xr-x 2 user group    22 Apr 10 15:09 syslog
    user@host:~/slaacer$

## Installation

There are four components that need to be configured:

 * SQL server
 * central syslog server
 * port mirroring
 * the SLAACer daemon

Throughout, we will assume that:

 * your central syslog server you administrate lives at '1.2.3.4'
 * the SQL server is at 'sql.example.com'

**N.B.** make sure you are careful to replace all the placeholders ('%...%')

### SQL

This should be just a simple case of just replacing the place holders '%DATABASE%', '%USERNAME%' and '%PASSWORD%' in the included 'sql' directory for your particular SQL server and loading it in.

### PostgreSQL
    
    sql:~# cat `<<'EOF' >`> /etc/postgresql/8.3/main/pg_hba.conf
    host    %DATABASE%    mac2addr    1.2.3.4/32     md5
    EOF
    
    sql:~# /etc/init.d/postgresql-8.3 reload
    
    sql:~# su - postgres
    postgres@sql:~$ psql -f /path/to/slaacer/sql/pqsql %DATABASE%

### syslog

Like for the SQL case, it should be just a simple case of replacing the place holders '%SQL-SERVER%', '%DATABASE%', '%USERNAME%' and '%PASSWORD%' in the included 'syslog' directory.

### syslog-ng

Make sure you have the 'psql' (Debian package 'postgresql-client') tool installed and that you insert the 'syslog/syslog-ng' file into your syslog-ng configuration (additionally you might wish to add 'syslog/syslog-ng.ipv4-dhcp-hook' to record you IPv4 allocations too).

    syslog:~# cat `<<'EOF' >`> /etc/syslog-ng/pgpass
    sql.example.com:5432:%DATABASE%:mac2addr:%PASSWORD%
    EOF
    
    syslog:~# chown root:root /etc/syslog-ng/pgpass
    syslog:~# chmod 600 /etc/syslog-ng/pgpass
    
    syslog:~# /etc/init.d/syslog-ng reload

### Port Mirror

This section is more than likely the most difficult part and so to try and help out I have broken it up into two chunks.  The "quick'n'dirty" approach that gets you going without any thought about the quantity of traffic flooding your LAN but should just work, the second part lists constraints and approaches to make you feel less dirty in regards to the deployment.

**N.B.** as a helpful aid to track bandwidth usage on your LAN, I strongly recommend [Torrus](http://torrus.org/).  It is trivial to set up and it gives you RRD graphs for *every* switchport on your network.

### Cisco

You need to create a single global VLAN than spans your entire LAN (in our example we shall use VLAN ID '300').  The VLAN should be trunked up to every switch stack and you will need to type the following on either your VTP master, or each of your switch stacks:

    vlan 300
      name v6span-nd
      remote-span

At all your L3 router source(s)((If you have your IP addresses assigned at your core routers ('L2 to the edge'), then you RSPAN source on those core routers.  If you have your IP addressing at your access layer ('L3 to the edge') then you need to RSPAN source from there, yes 48 ports per switch is what we have to do, but it works)), assuming all your IPv6 traffic comes off VLAN 33, we want to only capture ingress (received) traffic:

    monitor session 8 source vlan 33 rx
    monitor session 8 destination remote vlan 300

**N.B.** if you are fortunate enough to have 3750X/3560X's at your access layer, then you can use [FRSPAN (Filter RSPAN)](http://www.cisco.com/en/US/docs/switches/lan/catalyst3750x_3560x/software/release/12.2_55_se/configuration/guide/swspan.html#wp1334887) to capture *just* then neighbour discovery traffic.  If you be a simple case of just adding 'monitor session 8 filter ipv6 ...' and the necessary access list.

On the switch that the SLAACer server connects to (assuming it's monitoring port, 'eth1' is on 'Gi1/0/1'), type:

    monitor session 8 source remote vlan 300
    monitor session 8 destination interface Gi1/0/1

## Improving

Remote SPAN'ing functions by creating a VLAN that has MAC learning *disabled*, this unfortunately means that every packet is turned into a broadcast packet, this becomes a nasty problem as the number of edge switch stacks increase.  You can control the problem by creating a unique RSPAN VLAN for each edge switchstack which would effectively uni-directionalise the traffic.

The most powerful feature you need to look into is VACL's which enable you to apply filters to the RSPAN VLAN to control what traffic is actually passed around.  There are a number of unfortunate appliable constraints when dealing with VACL's and IPv6 traffic:

 * C3750 will not VACL RSPAN sources

 * C3750/C6500 [will not VACL with IPv6 ACLs](http://www.cisco.com/en/US/tech/tk389/tk814/technologies_configuration_example09186a00808122ac.shtml#vacl)

 * C3750 will not VACL IPv6 Ethernet frames with MAC ACLs

 * C6500 with even a single DFC3A will not VACL Ethernet frames with MAC ACLs (no 'mac classify-packet' command)

 * IOS versions pre-12.2(55) on a C3750 are affected by CSCtd72626 where an RSPAN does not include multicast IPv6 traffic (MAC address destination '33:33:ww:xx:yy:zz') that prevents catching NS-DAD packets

My [organisation's](http://www.soas.ac.uk/) network is made up of C3750's at the edge (configured in L3 'at the edge') feeding back to two C6500's (with DFC3A modules) using port-channels.  The SLAACer daemon sits on a Linux box hanging off a C3750E that pipes back to the C6500 core.  With this topology some of those contraints are a royal pain...fortunately at peak most of those port-channels peak at around 5% utilisation.

So far, what I am currently using is the following (the C6500 will prune all IPv4 traffic whilst the C3750E prunes off the ARP traffic):

    6500#show ip access-lists slaacer-v4
    Extended IP access list slaacer-v4
        10 permit ip any any
    
    6500#show vlan access-map slaacer
    Vlan access-map "slaacer"  10
            match: ip address slaacer-v4
            action: drop
    
    6500#show vlan filter
    VLAN Map slaacer:
            Configured on VLANs:  300
                Active on VLANs:  300

    
    3750E#show access-lists slaacer
    
    Extended MAC access list slaacer
        permit any any lsap 0xAAAA 0x0
        permit any any lsap 0x4242 0x0
    
    3750E#show vlan access-map slaacer
    Vlan access-map "slaacer"  10
      Match clauses:
        mac address: slaacer
      Action:
        forward
    Vlan access-map "slaacer"  20
      Match clauses:
      Action:
        drop
    
    3750E#show vlan filter
    VLAN Map slaacer is filtering VLANs:
      300

#### Possible IPv6/MAC VACL Version
    
    no vlan filter slaacer vlan-list 902
    no vlan access-map slaacer
    no mac access-list extended slaacer
    
    mac access-list extended slaacer
            deny any any 0x4242 0x0
            deny any any 0xaaaa 0x0
            deny any 3333.0000.0000 0000.ffff.ffff 0x86dd 0x0
            permit any any
    vlan access-map slaacer 10
            match mac address slaccer
            action drop
    vlan access-map slaacer 20
            action forward
    
    vlan filter slaacer vlan-list 902
    ----------------
    ipv6 access-list slaacer-v6
            permit icmp FF80::/64 host FF02::2 sequence 10
            permit icmp host :: FF02::1:0:0/96 sequence 20
    ip access-list extended slaacer-v4
            permit ip any any
    
    vlan access-map slaacer 10
            match ip address slaacer-v4
            action drop
    
    vlan filter slaacer vlan-list 902

### SLAACer

'slaacer' is placed on the system that is the destination for all the port mirror'ed traffic you are collecting (lets assume the interface is 'eth1').  We first need to configure the system to send the 'slaacer' generated syslog traffic to your central syslog server.  We do this, assuming you run 'rsyslog' on the system, by adding the following to send the traffic over TCP (if you want UDP, replace '@@' with '@'):

    root@slaacer:~# cat `<<'EOF' >` /etc/rsyslog.d/local-slaacer.conf
    if $programname == 'slaacer' then @@1.2.3.4
    & ~
    EOF
    
    root@slaacer:~# /etc/init.d/rsyslog reload

For now, initially call 'slaacer' in a [screen](http://www.kuro5hin.org/story/2004/3/9/16838/14935) session:
    
    # slaacer -i eth1 -n

'slaacer' will now parse traffic coming in over 'eth1' and print to stderr (as well as syslog) a message describing the traffic it sees.  The syslog messages will be processed by your central syslog server and the SQL table 'mac2addr' will be updated with this live information.
