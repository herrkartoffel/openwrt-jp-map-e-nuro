# **Configuring OpenWrt to work with Japan NURO IPv6 (MAP-E) and your OWN GPON device**

## **Assumptions:**

1. You use your own GPON device (discussion [here](https://github.com/meh301/HG8045Q/issues/2))
2. You have setup VLAN10 on your WAN/GPON and your router receives a /56 delegated prefix with PD
3. The WAN interface on your router is assigned a /64 via DHCPv6
4. You have the map package installed (opkg install map), if it is not available as a interface type please reboot the router

## **MAP-E configuration:**

Create a new MAP-E interface, create a new interface and name it (e.g. _WAN6MAPE_), and fill the parameters:

*   If your IPv6 address starts with 240d:000f:**0**, the parameters are:
    *   Protocol: MAP/LW4over6
    *   Type: MAP-E
    *   BR/DMR/AFTR: 2001:3b8:200:ff9::1
    *   IPv4 prefix: **219.104.128.0**
    *   IPv4 prefix length: 20
    *   IPv6 prefix: **240d:000f:0000::**
    *   IPv6 prefix length: 36
    *   EA-bit length: 20
    *   PSID-bits length: 8
    *   PSID offset: 4
    *   From advanced settings, make sure it has **WAN6 as Tunnel Link**, and check the box **Use legacy MAP**, and set the MTU to 1460

*   If your IPv6 address starts with 240d:000f:**1**, the parameters are:
    *   Protocol: MAP/LW4over6
    *   Type: MAP-E
    *   BR/DMR/AFTR: 2001:3b8:200:ff9::1
    *   IPv4 prefix: **219.104.144.0**
    *   IPv4 prefix length: 20
    *   IPv6 prefix: **240d:000f:1000::**
    *   IPv6 prefix length: 36
    *   EA-bit length: 20
    *   PSID-bits length: 8
    *   PSID offset: 4
    *   From advanced settings, make sure it has **WAN6 as Tunnel Link**, and check the box **Use legacy MAP**, and set the MTU to 1460

*   If your IPv6 address starts with 240d:000f:**2**, the parameters are:
    *   Protocol: MAP/LW4over6
    *   Type: MAP-E
    *   BR/DMR/AFTR: 2001:3b8:200:ff9::1
    *   IPv4 prefix: **219.104.160.0**
    *   IPv4 prefix length: 20
    *   IPv6 prefix: **240d:000f:2000::**
    *   IPv6 prefix length: 36
    *   EA-bit length: 20
    *   PSID-bits length: 8
    *   PSID offset: 4
    *   From advanced settings, make sure it has **WAN6 as Tunnel Link**, and check the box **Use legacy MAP**, and set the MTU to 1460

*   If your IPv6 address starts with 240d:000f:**3**, the parameters are:
    *   Protocol: MAP/LW4over6
    *   Type: MAP-E
    *   BR/DMR/AFTR: 2001:3b8:200:ff9::1
    *   IPv4 prefix: **219.104.176.0**
    *   IPv4 prefix length: 20
    *   IPv6 prefix: **240d:000f:3000::**
    *   IPv6 prefix length: 36
    *   EA-bit length: 20
    *   PSID-bits length: 8
    *   PSID offset: 4
    *   From advanced settings, make sure it has **WAN6 as Tunnel Link**, and check the box **Use legacy MAP**, and set the MTU to 1460

![WANMAP](https://github.com/herrkartoffel/openwrt-jp-map-e-nuro/blob/main/wanmap.png?raw=true)

![WANMAP-Advanced](https://github.com/herrkartoffel/openwrt-jp-map-e-nuro/blob/main/wanmap-advanced.png?raw=true)

Note: Don't forget to add this _WAN6MAPE_ interface to same firewall zone as WAN/WAN6 since this is also part of WAN.

## PING/EXHAUSTED PORTS FIX

MAP-E with IPv4 sharing from ISP is designed to share same IPv4 address with many customers, with different ports being assigned based on IETF rules, the above linked parameter calculator already shown the assigned ports, usually it's divided into groups of 16 ports, according to [this discussion](https://github.com/fakemanhk/openwrt-jp-ipoe/discussions/10) JPNE assigns 15 groups (240 ports), while NURO assign 63 groups (1008 ports). In most cases this should be enough for most home uses (since only IPv4 connections will use them), however a recent test with well known IPv4 based [website](https://nichiban.co.jp) that uses many sessions showing a significant lagging while loading. After investigation the OpenWrt firewall statistics indicating only first group of assigned ports (i.e. only 16 ports) being used and this is the reason of lagging when a large number of simultaneous IPv4 sessions opening, also IPv4 PING is not working. Not sure if it's because Japan ISP MAP-E configuration has something MAP package can't deal with, as a result a system change is required for _**/lib/netifd/proto/map.sh.**_ Below is the diff of original vs new map.sh:

```
root@OpenWrt# diff -c map.sh.old map.sh.new
*** map.sh.old  Sat Mar  4 03:57:30 2023
--- map.sh.new     Tue Mar  7 00:09:22 2023
***************
*** 13,18 ****
--- 13,40 ----
  # MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
  # GNU General Public License for more details.
  
+ #------------------------------------
+ #     Modifications by 'cmspam'
+ #------------------------------------
+ # These modifications are made to ensure that the multiple port ranges
+ # used with Japan's MAP-E implementation are actually SNAT-ed to correctly.
+ 
+ #------------------------------------
+ #     Setting to not SNAT to certain ports
+ #------------------------------------
+ # This is a space-delimited list of ports to not SNAT to.
+ # If, for example, you use some ports for servers, it's best to
+ # include them here, so that they are not used with SNAT.
+ # Port ranges are not supported, so please list each port individually
+ # separated with a space.
+ # 
+ # Example, if you want to not SNAT to 2938, 7088, and 10233
+ #
+ #DONT_SNAT_TO="2938 7088 10233"
+ 
+ DONT_SNAT_TO="0"
+ 
+ 
  [ -n "$INCLUDE_ONLY" ] || {
        . /lib/functions.sh
        . /lib/functions/network.sh
***************
*** 140,158 ****
              json_add_string snat_ip $(eval "echo \$RULE_${k}_IPV4ADDR")
            json_close_object
          else
            for portset in $(eval "echo \$RULE_${k}_PORTSETS"); do
!               for proto in icmp tcp udp; do
!               json_add_object ""
!                 json_add_string type nat
!                 json_add_string target SNAT
!                 json_add_string family inet
!                 json_add_string proto "$proto"
!                   json_add_boolean connlimit_ports 1
!                   json_add_string snat_ip $(eval "echo \$RULE_${k}_IPV4ADDR")
!                   json_add_string snat_port "$portset"
!               json_close_object
!               done
            done
          fi
          if [ "$maptype" = "map-t" ]; then
                [ -z "$zone" ] && zone=$(fw3 -q network $iface 2>/dev/null)
--- 162,233 ----
              json_add_string snat_ip $(eval "echo \$RULE_${k}_IPV4ADDR")
            json_close_object
          else
+ 
+ #------------------------------------
+           #MODIFICATION 1: Get all ports, and full port count.
+           #                Don't include ports in DONT_SNAT_TO
+ #------------------------------------
+           local portcount=0
+           local allports=""
            for portset in $(eval "echo \$RULE_${k}_PORTSETS"); do
!               local startport=$(echo $portset | cut -d'-' -f1)
!               local endport=$(echo $portset | cut -d'-' -f2)
!               for x in $(seq $startport $endport); do
!                       if ! echo "$DONT_SNAT_TO" | tr ' ' '\n' | grep -qw $x; then                        
!                               allports="$allports $portcount : $x , "
!                               portcount=`expr $portcount + 1`
!                       fi
!               done
            done
+           allports=${allports%??}
+ #------------------------------------
+           #END MODIFICATION 1
+ #------------------------------------
+           
+ #------------------------------------
+             #MODIFICATION 2: Create mape table
+ #------------------------------------
+             nft add table inet mape
+             nft add chain inet mape srcnat {type nat hook postrouting priority 0\; policy accept\; }
+ #------------------------------------
+           #END MODIFICATION 2
+ #------------------------------------
+ 
+ 
+ #------------------------------------
+           #MODIFICATION 3: Create the rules to snat to all the ports
+ #------------------------------------
+           local counter=0
+           
+             for proto in icmp tcp udp; do
+                 nft add rule inet mape srcnat ip protocol $proto oifname "map-$cfg" snat ip to $(eval "echo \$RULE_${k}_IPV4ADDR") : numgen inc mod $portcount map { $allports }
+             done
+           #END MODIFICATION 3
+           
+ #------------------------------------
+           #MODIFICATION 4: Comment out original SNAT implementation.
+ #------------------------------------
+ #            for portset in $(eval "echo \$RULE_${k}_PORTSETS"); do
+ #              for proto in icmp tcp udp; do
+ #                json_add_object ""
+ #                  json_add_string type nat
+ #                  json_add_string target SNAT
+ #                  json_add_string family inet
+ #                  json_add_string proto "$proto"
+ #                  json_add_boolean connlimit_ports 1
+ #                  json_add_string snat_ip $(eval "echo \$RULE_${k}_IPV4ADDR")
+ #                  json_add_string snat_port "$portset"
+ #                json_close_object
+ #              done
+ #            done
+ #------------------------------------
+           #END MODIFICATION 4
+ #-------------------------------------
+ #------------------------------------
+           #ALL MODIFICATIONS FINISHED
+ #------------------------------------
+ 
+ 
          fi
          if [ "$maptype" = "map-t" ]; then
                [ -z "$zone" ] && zone=$(fw3 -q network $iface 2>/dev/null)
```

You can also download the whole file [here](https://github.com/fakemanhk/openwrt-jp-ipoe/blob/main/map.sh.new), _**don't forget to turn on the execute bit (chmod +x)**_ of the file after replacement.

After editing, please restart IPv6 interface, or simply reboot router, you'll see that IPv4 PING is working as well as observing more port groups passing traffic now.

## Final result:

![WAN](https://github.com/herrkartoffel/openwrt-jp-map-e-nuro/blob/main/wan.png?raw=true)


### Reference materials:

Credits to [fakemanhk's guide](https://github.com/fakemanhk/openwrt-jp-ipoe) which helped with the ping/port issue.

[https://github.com/site-u2023/config-software/blob/main/map-e-nuro.sh]

[https://github.com/fakemanhk/openwrt-jp-ipoe]

_First draft: 18 April 2024_
