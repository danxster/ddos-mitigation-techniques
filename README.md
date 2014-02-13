ddos-mitigation-techniques
==========================

Dns:

#create a new table
iptables -N RATELIMITER

# have that drop requests when they start getting too frequent
iptables -I RATELIMITER -m hashlimit    --hashlimit-name DNS --hashlimit-above 50/minute --hashlimit-mode srcip    --hashlimit-burst 100 --hashlimit-srcmask 28 -j DROP

# send all incoming requests through that table
iptables -I INPUT -p udp --dport 53 -j RATELIMITER

*******below does noes work for icmp traffic*******
**this is the most popular method**

iptables -A INPUT -p udp --dport 53 --set --name dnslimit
iptables -A INPUT -p udp --dport 53 -m recent --update --seconds 60 --hitcount 11 --name dnslimit -j DROP

Webserver:

iptables -F
iptables -X
iptables -N ATTACKED
iptables -N ATTK_CHECK
iptables -N SYN_FLOOD
iptables -A INPUT -p tcp ! --syn -m state --state NEW -j DROP
iptables -A INPUT -f -j DROP
iptables -A INPUT -p tcp --tcp-flags ALL ALL -j DROP
iptables -A INPUT -p tcp --tcp-flags ALL NONE -j DROP
iptables -A INPUT -p tcp --syn -j SYN_FLOOD
iptables -A SYN_FLOOD -p tcp --syn -m hashlimit --hashlimit 100/sec --hashlimit-burst 3 --hashlimit-htable-expire 3600 --hashlimit-mode srcip  --hashlimit-name synflood -j ACCEPT
iptables -A SYN_FLOOD -j ATTK_CHECK
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -p tcp -m tcp --dport 80 -m recent --update --seconds 1800 --name BANNED --rsource -j DROP
iptables -A INPUT -p tcp -m tcp --dport 80 -m state --state NEW -j ATTK_CHECK
iptables -A ATTACKED -m limit --limit 5/min -j LOG --log-prefix "IPTABLES (Rule ATTACKED): " --log-level 7
iptables -A ATTACKED -m recent --set --name BANNED --rsource -j DROP
iptables -A ATTK_CHECK -m recent --set --name ATTK
iptables -A ATTK_CHECK -m recent --update --seconds 180 --hitcount 20 --name ATTK --rsource -j ATTACKED
iptables -A ATTK_CHECK -m recent --update --seconds 60 --hitcount 6 --name ATTK --rsource -j ATTACKED
iptables -A ATTK_CHECK -j ACCEPT
