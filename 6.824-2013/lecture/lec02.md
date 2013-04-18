# Lecture 2: Infrastructure: RPC and threads

## Remote Procedure Call (RPC)

* a key piece of distrib sys machinery; you'll see in lab 1
* goal: easy-to-program network communication
	* hides most details of client/server communication
	* client call is much like ordinary procedure call
	* server handlers are much like ordinary procedures

> **RPC is widely used!**
> **RPC ideally makes net communication look just like fn call**
> **RPC aims for this level of transparency**

### RPC message diagram:

		Client  Server
		request--->
		<---response

### Software structure
		client app         handlers
		stubs           dispatcher
		RPC lib           RPC lib
		net  ------------ net



### RPC problem: what to do about failures?

* e.g. lost packets, broken network, crashed servers


### Simplest scheme: "at least once" behavior

### Better RPC behavior: "at most once"

### What about "exactly once"?

* at-most-once plus unbounded retries


###Go RPC is "at-most-once"
