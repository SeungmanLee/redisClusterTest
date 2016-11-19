# redisClusterTest
Redis cluster 구성 테스트


레디스는 설치되어있다고 치자.

https://www.youtube.com/watch?v=s4YpCA2Y_-Q&feature=share


ruby 도 설치되어있고, ruby redis 도 설치되어있다고 치자.

<pre>
sudo apt-get install gem

gem install redis
</pre>

준비
--------

create-cluster start 를 이용하여   127.0.0.1 : 30001 ~ 30006 까지 총 6개의 instance 가 실행된 상태에서 테스트를 시작한다.

<pre>
utils/create-cluster/create-cluster start
</pre>

구성
----

master : 1, 2, 3

slave : 4, 5, 6

=> [ 1, 4 ] , [ 2, 5 ] , [ 3 , 6 ]  총 3개의 물리서버로 구분된 상태.  각 서버별로 2개 instance 를 구동한 것으로 가정한다.

master - slave 구성

2 <- 4

3 <- 5

1 <- 6

이제 시작이다
--

* master 를 구성한다.  
복제본 설정은 하지 않는다.  
이 경우 모든 노드가 master 가 된다.
slot 은 redis-trib 가 알아서 분배할 것이다.

<pre>
smlee@ubuntu-server:~/redis-3.2.5/src$ ./redis-trib.rb create 127.0.0.1:30001 127.0.0.1:30002 127.0.0.1:30003

>>> Creating cluster
>>> Performing hash slots allocation on 3 nodes...
Using 3 masters:
127.0.0.1:30001
127.0.0.1:30002
127.0.0.1:30003

M: b01033c87010c60c29564fdb692181175126741f 127.0.0.1:30001
  slots:0-5460 (5461 slots) master
M: acb8e8d001285f669aa9f263c86cc3029cab14aa 127.0.0.1:30002
  slots:5461-10922 (5462 slots) master
M: 31bf1057ce5d38d76d6765fbb230d5e3834b9c5a 127.0.0.1:30003
  slots:10923-16383 (5461 slots) master
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join..
>>> Performing Cluster Check (using node 127.0.0.1:30001)
M: b01033c87010c60c29564fdb692181175126741f 127.0.0.1:30001
  slots:0-5460 (5461 slots) master
  0 additional replica(s)
M: acb8e8d001285f669aa9f263c86cc3029cab14aa 127.0.0.1:30002
  slots:5461-10922 (5462 slots) master
  0 additional replica(s)
M: 31bf1057ce5d38d76d6765fbb230d5e3834b9c5a 127.0.0.1:30003
  slots:10923-16383 (5461 slots) master
  0 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
smlee@ubuntu-server:~/redis-3.2.5/src$
</pre>


* cluster 상태를 확인해보자.
master 3개에 slot 이 1/3 씩 할당되어있다.

<pre>
smlee@ubuntu-server:~/redis-3.2.5/src$ redis-cli -c -p 30001 cluster nodes

b01033c87010c60c29564fdb692181175126741f 127.0.0.1:30001 myself,master - 0 0 1 connected 0-5460
acb8e8d001285f669aa9f263c86cc3029cab14aa 127.0.0.1:30002 master - 0 1479550894031 2 connected 5461-10922
31bf1057ce5d38d76d6765fbb230d5e3834b9c5a 127.0.0.1:30003 master - 0 1479550894031 3 connected 10923-16383
smlee@ubuntu-server:~/redis-3.2.5/src$
</pre>


* slave 들을 cluster 에 join 시켜보자.
4번 노드를 cluster 로 조인하고, cluster 의 상태를 확인하면 된다.


<pre>
smlee@ubuntu-server:~/redis-3.2.5/src$ redis-cli -c -p 30004 cluster meet 127.0.0.1 30001

OK
smlee@ubuntu-server:~/redis-3.2.5/src$ redis-cli -c -p 30001 cluster nodes

65d45459e0a0da3f9f2ff959e072011181d1777f 127.0.0.1:30004 master - 0 1479550930780 0 connected
b01033c87010c60c29564fdb692181175126741f 127.0.0.1:30001 myself,master - 0 0 1 connected 0-5460
acb8e8d001285f669aa9f263c86cc3029cab14aa 127.0.0.1:30002 master - 0 1479550930780 2 connected 5461-10922
31bf1057ce5d38d76d6765fbb230d5e3834b9c5a 127.0.0.1:30003 master - 0 1479550930780 3 connected 10923-16383
smlee@ubuntu-server:~/redis-3.2.5/src$
</pre>


* 나머지 5, 6번도 join 하자.

<pre>

smlee@ubuntu-server:~/redis-3.2.5/src$
smlee@ubuntu-server:~/redis-3.2.5/src$ redis-cli -c -p 30004 cluster meet 127.0.0.1 30005

OK
smlee@ubuntu-server:~/redis-3.2.5/src$ redis-cli -c -p 30004 cluster meet 127.0.0.1 30006

OK
smlee@ubuntu-server:~/redis-3.2.5/src$ redis-cli -c -p 30001 cluster nodes

31bf1057ce5d38d76d6765fbb230d5e3834b9c5a 127.0.0.1:30003 master - 0 1479550946967 3 connected 10923-16383
acb8e8d001285f669aa9f263c86cc3029cab14aa 127.0.0.1:30002 master - 0 1479550946967 2 connected 5461-10922
b01033c87010c60c29564fdb692181175126741f 127.0.0.1:30001 myself,master - 0 0 1 connected 0-5460
65d45459e0a0da3f9f2ff959e072011181d1777f 127.0.0.1:30004 master - 0 1479550946967 4 connected
d064515620fd1681a115e782e751b026b7124a2a 127.0.0.1:30006 master - 0 1479550947066 0 connected
8388c4ddfaca4c3505ce348e59479d019823e2d8 127.0.0.1:30005 master - 0 1479550946967 5 connected

</pre>


이제부터 복제본 구성이다.
--

* 4번 노드는 master 2번의 slave 가 될 것이다.

<pre>
smlee@ubuntu-server:~/redis-3.2.5/src$ redis-cli -c -p 30004 cluster replicate acb8e8d001285f669aa9f263c86cc3029cab14aa

OK
smlee@ubuntu-server:~/redis-3.2.5/src$ redis-cli -c -p 30001 cluster nodes

31bf1057ce5d38d76d6765fbb230d5e3834b9c5a 127.0.0.1:30003 master - 0 1479551114472 3 connected 10923-16383
acb8e8d001285f669aa9f263c86cc3029cab14aa 127.0.0.1:30002 master - 0 1479551114472 2 connected 5461-10922
b01033c87010c60c29564fdb692181175126741f 127.0.0.1:30001 myself,master - 0 0 1 connected 0-5460
65d45459e0a0da3f9f2ff959e072011181d1777f 127.0.0.1:30004 slave acb8e8d001285f669aa9f263c86cc3029cab14aa 0 1479551114472 4 connected
d064515620fd1681a115e782e751b026b7124a2a 127.0.0.1:30006 master - 0 1479551114581 0 connected
8388c4ddfaca4c3505ce348e59479d019823e2d8 127.0.0.1:30005 master - 0 1479551114472 5 connected
smlee@ubuntu-server:~/redis-3.2.5/src$

</pre>

* 5번 노드는 master 3번의 slave 가 될것이다.

<pre>

smlee@ubuntu-server:~/redis-3.2.5/src$ redis-cli -c -p 30005 cluster replicate 31bf1057ce5d38d76d6765fbb230d5e3834b9c5a

OK
smlee@ubuntu-server:~/redis-3.2.5/src$ redis-cli -c -p 30001 cluster nodes

31bf1057ce5d38d76d6765fbb230d5e3834b9c5a 127.0.0.1:30003 master - 0 1479551142069 3 connected 10923-16383
acb8e8d001285f669aa9f263c86cc3029cab14aa 127.0.0.1:30002 master - 0 1479551142069 2 connected 5461-10922
b01033c87010c60c29564fdb692181175126741f 127.0.0.1:30001 myself,master - 0 0 1 connected 0-5460
65d45459e0a0da3f9f2ff959e072011181d1777f 127.0.0.1:30004 slave acb8e8d001285f669aa9f263c86cc3029cab14aa 0 1479551142069 4 connected
d064515620fd1681a115e782e751b026b7124a2a 127.0.0.1:30006 master - 0 1479551142170 0 connected
8388c4ddfaca4c3505ce348e59479d019823e2d8 127.0.0.1:30005 slave 31bf1057ce5d38d76d6765fbb230d5e3834b9c5a 0 1479551142069 5 connected
smlee@ubuntu-server:~/redis-3.2.5/src$

</pre>

* 마지막으로 6번 노드는 1번의 slave 가 될 것이다.

<pre>

smlee@ubuntu-server:~/redis-3.2.5/src$
smlee@ubuntu-server:~/redis-3.2.5/src$ redis-cli -c -p 30006 cluster replicate b01033c87010c60c29564fdb692181175126741f

OK
smlee@ubuntu-server:~/redis-3.2.5/src$ redis-cli -c -p 30001 cluster nodes

31bf1057ce5d38d76d6765fbb230d5e3834b9c5a 127.0.0.1:30003 master - 0 1479551167574 3 connected 10923-16383
acb8e8d001285f669aa9f263c86cc3029cab14aa 127.0.0.1:30002 master - 0 1479551167574 2 connected 5461-10922
b01033c87010c60c29564fdb692181175126741f 127.0.0.1:30001 myself,master - 0 0 1 connected 0-5460
65d45459e0a0da3f9f2ff959e072011181d1777f 127.0.0.1:30004 slave acb8e8d001285f669aa9f263c86cc3029cab14aa 0 1479551167574 4 connected
d064515620fd1681a115e782e751b026b7124a2a 127.0.0.1:30006 slave b01033c87010c60c29564fdb692181175126741f 0 1479551167574 1 connected
8388c4ddfaca4c3505ce348e59479d019823e2d8 127.0.0.1:30005 slave 31bf1057ce5d38d76d6765fbb230d5e3834b9c5a 0 1479551167574 5 connected

smlee@ubuntu-server:~/redis-3.2.5/src$
smlee@ubuntu-server:~/redis-3.2.5/src$

</pre>


* 노드 구성은 완료되었다. 핵심은 동일 서버에 master 와 slave 가 구성되지 않는것이 포인트인데..
redis-trib 로 복제본(replica) 갯수를 1로 지정하면 내맘같지 않게 구성된다.

한번 해보면 그닥 어렵지 않다. 한번도 안해보면 어렵다.
