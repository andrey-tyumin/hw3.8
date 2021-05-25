### hw3.8
1. ipvs. Если при запросе на VIP сделать подряд несколько запросов  
(например, for i in {1..50}; do curl -I -s 172.28.128.200>/dev/null; done ),  
ответы будут получены почти мгновенно. Тем не менее, в выводе ipvsadm -Ln еще  
некоторое время будут висеть активные InActConn. Почему так происходит?  

Перед закрытием соединения, оно переодит в состояние  
TIME-WAIT = 2MSL(maximum segment lifetime - время ожидания перед закрытием соединения(по RFC1122=2 min).  
MSL - макс. время в течении которого дейтаграмма IP может оставаться в сети. 

---

