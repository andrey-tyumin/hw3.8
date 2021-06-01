### hw3.8
1. ipvs. Если при запросе на VIP сделать подряд несколько запросов  
(например, for i in {1..50}; do curl -I -s 172.28.128.200>/dev/null; done ),  
ответы будут получены почти мгновенно. Тем не менее, в выводе ipvsadm -Ln еще  
некоторое время будут висеть активные InActConn. Почему так происходит?  

Перед закрытием соединения, оно переодит в состояние  
TIME-WAIT = 2MSL(maximum segment lifetime - время ожидания перед закрытием соединения(по RFC1122=2 min).  
MSL - макс. время в течении которого дейтаграмма IP может оставаться в сети.  
Пояснение:  
Пункт 4.11 LVS-HOWTO: http://www.austintek.com/LVS/LVS-HOWTO/HOWTO/LVS-HOWTO.ipvsadm.html#ActiveConn  
В выводе ipvsadm присутствуют следующие статусы соединений:  

   > ActiveConn - in ESTABLISHED state  
   > InActConn - any other state   

т.е. соединение в состоянии TIME-WAIT будет отображено в колонке InActConn.  

Эти соединения(в нашем случае) будут отображаться в течении времени timeout(TIME-WAIT).  
В случае когда все хорошо:  
> Services like http (in non-persistent i.e. HTTP /1.0 mode) or ftp-data(port 20) which close the connections as soon as the hit/data (html page, or gif etc) has >been retrieved (<1sec). You're unlikely to see anything in the ActiveConn column with these LVS'ed services. You'll see an entry in the InActConn column untill >the connection times out. If you're getting 1000connections/sec and it takes 60secs for the connection to time out (the normal timeout), then you'll have 60,000 >InActConns. This number of InActConn is quite normal. If you are running an e-commerce site with 300secs of persistence, you'll have 300,000 InActConn entries. >Each entry takes 128bytes (300,000 entries is about 40M of memory, make sure you have enough RAM for your application). The number of ActiveConn might be very >small. 


---

2. На лекции мы познакомились отдельно с ipvs и отдельно с keepalived.  
Воспользовавшись этими знаниями, совместите технологии вместе  
(VIP должен подниматься демоном keepalived). Приложите конфигурационные файлы,  
которые у вас получились, и продемонстрируйте работу получившейся конструкции.  
Используйте для директора отдельный хост, не совмещая его с риалом! Подобная схема  
возможна, но выходит за рамки рассмотренного на лекции.  

Vagrantfile:
```
# -*- mode: ruby -*-
# vi: set ft=ruby :

balancers= {
  'balancer1' => '10',
  'balancer2' => '20',
}

servers = {
  'server1' => '30',
  'server2' => '40',
}

Vagrant.configure("2") do |config|
  config.vm.network "private_network", virtualbox__intnet: true, auto_config: false
  config.vm.box = "bento/ubuntu-20.04"
#  config.vm.provider :virtualbox do |v|
#  v.customize ["modifyvm", :id, "--memory", 512]
#  end
#servers set
  servers.each do |k, v|
    config.vm.define k do |servers|
      servers.vm.provision "shell" do |s|
        s.inline = "hostname $1;"\
          "ip addr add $2 dev eth1;"\
          "ip link set dev eth1 up;"\
          "apt -y update;"\
          "apt -y upgrade;"\
          "apt -y install nginx;"\
	  "ip addr add 172.28.128.200/32 dev lo label lo:200;"\
          "sysctl -w net.ipv4.conf.all.arp_ignore=1;"\
          "sysctl -w net.ipv4.conf.all.arp_announce=2"
        s.args = [k, "172.28.128.#{v}/24"]
      end
    end
   end
#balancers set
  balancers.each do |k, v|
    config.vm.define k do |balancers|
      balancers.vm.provision "shell" do |s|
        s.inline = "hostname $1;"\
          "ip addr add $2 dev eth1;"\
          "ip link set dev eth1 up;"\
          "apt -y update;"\
          "apt -y upgrade;"\
          "apt -y install keepalived ipvsadm;"\
	  "ipvsadm -A -t 172.28.128.200:80 -s rr;"\
	  "ipvsadm -a -t 172.28.128.200:80 -r 172.28.128.30:80 -g -w 1;"\
	  "ipvsadm -a -t 172.28.128.200:80 -r 172.28.128.40:80 -g -w 1"
        s.args = [k, "172.28.128.#{v}/24"]
      end
    end
   end
#client
  config.vm.define "client" do |client|
   client.vm.hostname = "client"
   client.vm.network "private_network", ip:"172.28.128.50",
   virtualbox__intnet: true
  end
#end machines set

end
```
balancer1 keepalived.conf:
```
root@balancer1:/home/vagrant# cat /etc/keepalived/keepalived.conf 
vrrp_instance VI_1 {
state MASTER      
interface eth1    
virtual_router_id 33
priority 100
advert_int 1

virtual_ipaddress {       
172.28.128.200/24 dev eth1
}    
}
```
balanser2 keepalived.conf
```
root@balancer2:/home/vagrant# cat /etc/keepalived/keepalived.conf 
vrrp_instance VI_1 {
state BACKUP
interface eth1    
virtual_router_id 33
priority 50
advert_int 1

virtual_ipaddress {       
172.28.128.200/24 dev eth1
}    
}
```
Погасим eth1 на balancer1 и проверим переключение: 
```
root@balancer1:/home/vagrant# ip link set eth1 down
root@balancer1:/home/vagrant# cat /var/log/syslog | grep VI_1
May 25 12:31:18 vagrant Keepalived_vrrp[54462]: (VI_1) Entering BACKUP STATE (init)
May 25 12:31:22 vagrant Keepalived_vrrp[54462]: (VI_1) Entering MASTER STATE
May 25 12:40:45 vagrant Keepalived_vrrp[54462]: (VI_1) Entering FAULT STATE
May 25 12:40:45 vagrant Keepalived_vrrp[54462]: (VI_1) sent 0 priority
---
root@balancer2:/home/vagrant# cat /var/log/syslog | grep VI_1
May 25 12:31:43 vagrant Keepalived_vrrp[54389]: (VI_1) Entering BACKUP STATE (init)
May 25 12:40:48 vagrant Keepalived_vrrp[54389]: (VI_1) Entering MASTER STATE
---
vagrant@client:~$ for i in {1..50}; do curl -I -s 172.28.128.200>/dev/null; done
---
root@balancer2:/home/vagrant# ipvsadm -Ln --stats
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port               Conns   InPkts  OutPkts  InBytes OutBytes
  -> RemoteAddress:Port
TCP  172.28.128.200:80                  50      300        0    19950        0
  -> 172.28.128.30:80                   25      150        0     9975        0
  -> 172.28.128.40:80                   25      150        0     9975        0
---
root@balancer1:/home/vagrant# ipvsadm -Ln --stats
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port               Conns   InPkts  OutPkts  InBytes OutBytes
  -> RemoteAddress:Port
TCP  172.28.128.200:80                   0        0        0        0        0
  -> 172.28.128.30:80                    0        0        0        0        0
  -> 172.28.128.40:80                    0        0        0        0        0
---
```
Работает balancer2.  
Поднимем eth1 на balancer1:
```
root@balancer1:/home/vagrant# ip link set eth1 up
root@balancer1:/home/vagrant# cat /var/log/syslog | grep VI_1
...
May 25 13:02:52 vagrant Keepalived_vrrp[54462]: (VI_1) Entering MASTER STATE
---
root@balancer2:/home/vagrant# cat /var/log/syslog | grep VI_1
May 25 12:31:43 vagrant Keepalived_vrrp[54389]: (VI_1) Entering BACKUP STATE (init)
May 25 12:40:48 vagrant Keepalived_vrrp[54389]: (VI_1) Entering MASTER STATE
root@balancer2:/home/vagrant# cat /var/log/syslog | grep VI_1
May 25 12:31:43 vagrant Keepalived_vrrp[54389]: (VI_1) Entering BACKUP STATE (init)
May 25 12:40:48 vagrant Keepalived_vrrp[54389]: (VI_1) Entering MASTER STATE
May 25 13:02:52 vagrant Keepalived_vrrp[54389]: (VI_1) Master received advert from 172.28.128.10 with higher priority 100, ours 50
May 25 13:02:52 vagrant Keepalived_vrrp[54389]: (VI_1) Entering BACKUP STATE
---
vagrant@client:~$ for i in {1..50}; do curl -I -s 172.28.128.200>/dev/null; done
---
root@balancer1:/home/vagrant# ipvsadm -Ln --stats
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port               Conns   InPkts  OutPkts  InBytes OutBytes
  -> RemoteAddress:Port
TCP  172.28.128.200:80                  50      300        0    19950        0
  -> 172.28.128.30:80                   25      150        0     9975        0
  -> 172.28.128.40:80                   25      150        0     9975        0
```
Работает balancer1.

---

3. В лекции мы использовали только 1 VIP адрес для балансировки. У такого подхода  
несколько отрицательных моментов, один из которых – невозможность активного  
использования нескольких хостов (1 адрес может только переехать с master на standby).  
Подумайте, сколько адресов оптимально использовать, если мы хотим без какой-либо  
деградации выдерживать потерю 1 из 3 хостов при входящем трафике 1.5 Гбит/с и  
физических линках хостов в 1 Гбит/с? Предполагается, что мы хотим задействовать  
3 балансировщика в активном режиме (то есть не 2 адреса на 3 хоста, один из которых  
в обычное время простаивает).  

3 балансировщика в активном режиме должны использавать минимум 3 адреса.  
При входящем трафике 1,5 Гбит/с отказ одного из балансировщиков не приведет  
к замедлению обслуживания запросов.  
Ответ: 3.  
