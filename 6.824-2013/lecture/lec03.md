# Lecture 3: Primary/Backup Replication

## Today's goals
* avaliablity
* correctness
* handle network failures and repair

### Tool: replication of service's state

### How to ensure two replicas(server) remain ientical?
1. copy the whole state, and/or
2. apply same operations in same order to both replicas

### what wider classes of failure would we like to handle?

* temporary or permanent loss of connectivity
* network partitions
* can't know if a server is crashed or just not reachable

## Lab2 goals
* tolerate network problems, including partition
    * either keep going, correctly
    * or suspend operations until network is repaired
* replacement of failed servers

### Lab2 overview

* agreement
    * "view server" decides who p and b are
    * clients and servers ask view server
    * they don't make independent decisions
    * only one vs, avoids multiple machines independently deciding who is p

* repair
	* view server can co-opt "idle" server as b after old b becomes p
	* primary initializes new backup's state

* the tricky part
	* only one primary!
	* primary must have state!

### View server
* maintains a sequence of "views"
* monitors server liveness

#### how to ensure new primary has up-to-date replica of state?
* only promote previous backup

#### how to avoid promoting a state-less backup?

lab 2 answer:
*    primary in each view must acknowledge that view to viewserver
*    viewserver must stay with current view until acknowledged
*    even if the primary seems to have failed
*    no point in proceeding since not acked == backup may not be initialized

#### how to ensure only one server acts as primary?
the rules:
1. non-backup must reject forwarded requests
2. primary in view i must have been primary or backup in view i-1
3. non-primary must reject direct client requests
4. primary must wait for backup to accept each request

#### how can new backup get state?

if S2 is backup in view i, but was not in view i-1, S2 should ask primary to transfer the complete state

rule for state transfer:
>  every operation (Lock,Unlock) must be either before or after state xfer
  if before, xferred state must reflect operation
  if after, primary must forward operation after xfer finishes


