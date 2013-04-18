# Lecture 1： Introduction and lab overview



What is a  distributed system?

>**multiple networked cooperating computers**
	

Why distribute?

* to **connect** physically separate entities
* to **achieve** security via physical isolation
* to **tolerate** faults via replication at separate sites
* to **increase** performance via parallel CPUs/mem/disk/net


But:

* complex, hard to debug
* advice: don't distribute if a central system will work

* * *
	
## MAIN TOPICS

### Topic: architecture

* Choice of interfaces  接口
* Single machine room or unified wide area system?  范围
* Client/server or peer-to-peer?   结构
* Interact w/ performance, security, fault behavior. 其他
	
### Topic: implementation

* Most systems organize distribution with some structuring framework(s)
* RPC, RMI, DSM, MapReduce, &c
	
### Topic: performance

* Distribution can hurt: network b/w and latency bottlenecks.
	  Lots of tricks, e.g. caching, threaded servers
* Distribution can help: parallelism, pick server near client
* Need a way to divide the load by N
	
### Topic: fault tolerance


### Topic: consistency

* * *

## LAB

### focus: fault tolerance and consistency -- central to distrib sys


### what fault-tolerance properties might we want?
	 
* available
* durable
* consistent
	
### what kinds of faults might we want to tolerate?

* network:
	* lost packets
	* duplicated packets
	* temporary network failure
		* server disconnected
		* network partitioned

* server
	* server crash+restart
	* server fails permanently
	* all servers fail simultaneously -- power/earthquake
	* bad case: crash mid-way through complex operation
	* bugs -- but not in this course
	* malice -- but not in this course
 
* client fails

### tools for dealing with faults?

* retry -- e.g. if pkt is lost, or server crash+restart
* replicate -- e.g. if  one server or part of net has failed
* replace -- for long-term health

* * *

## LAB1

### "failure model": single fail-stop failure

* ONLY failure: one server halts
* NO network failures
* NO re-start of servers
* thus: no response means that the server has halted
* thus: at least one of primary/backup guaranteed to be alive
* fail-stop reasonable for tightly coupled primary+backup
* fail-stop not a good model for biggish internet-based systems
	* due to network failures

### fault-tolerance scheme: replication via "primary/backup"
  
* replicate the service state
	* for each lock, locked or not
* one copy on primary server, one on backup server
* clients send operations to primary
* primary forwards to backup so backup can update its state
* primary replies to client after backup replies
