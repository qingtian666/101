### create network ns
```
mkdir -p /var/run/netns
find -L /var/run/netns -type l -delete
```
### start nginx docker with non network mode
```
docker run --network=none  -d nginx
```
### check corresponding pid
```
docker ps|grep nginx
docker inspect <containerid>|grep -i pid

"Pid": 876884,
"PidMode": "",
"PidsLimit": null,
```
### check network config for the container
```
nsenter -t 876884 -n ip a
```
### link network namespace
```
export pid=876884
ln -s /proc/$pid/ns/net /var/run/netns/$pid
ip netns list
```
### check docker bridge on the host 
```
brctl show
ip a
4: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:35:40:d3:8b brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:35ff:fe40:d38b/64 scope link
       valid_lft forever preferred_lft forever
```
### create veth pair
```
ip link add A type veth peer name B
```
### config A
```
brctl addif docker0 A
ip link set A up
```
### config B
```
SETIP=172.17.0.10
SETMASK=16
GATEWAY=172.17.0.1

ip link set B netns $pid
ip netns exec $pid ip link set dev B name eth0
ip netns exec $pid ip link set eth0 up
ip netns exec $pid ip addr add $SETIP/$SETMASK dev eth0
ip netns exec $pid ip route add default via $GATEWAY
```
### check connectivity
```
curl 172.17.0.10
```