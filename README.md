Download Link: https://assignmentchef.com/product/solved-cos418-assignment-5-key-value-storage-service
<br>
<h2><a id="user-content-introduction" class="anchor" href="https://github.com/theoliao1998/Distributed-Systems/tree/master/5%20Key-Value%20Storage%20Service#introduction" aria-hidden="true"></a>Introduction</h2>

In this assignment you will build a fault-tolerant key-value storage service using your Raft library from the previous assignments. Your key-value service will be structured as a replicated state machine with several key-value servers that coordinate their activities through the Raft log. Your key/value service should continue to process client requests as long as a majority of the servers are alive and can communicate, in spite of other failures or network partitions.

Your system will consist of clients and key/value servers, where each key/value server also acts as a Raft peer. Clients send <tt>Put()</tt>, <tt>Append()</tt>, and <tt>Get()</tt> RPCs to key/value servers (called kvraft servers), who then place those calls into the Raft log and execute them in order. A client can send an RPC to any of the kvraft servers, but if that server is not currently a Raft leader, or if there’s a failure, the client should retry by sending to a different server. If the operation is committed to the Raft log (and hence applied to the key/value state machine), its result is reported to the client. If the operation failed to commit (for example, if the leader was replaced), the server reports an error, and the client retries with a different server.

<h2><a id="user-content-software" class="anchor" href="https://github.com/theoliao1998/Distributed-Systems/tree/master/5%20Key-Value%20Storage%20Service#software" aria-hidden="true"></a>Software</h2>

We have supplied you with skeleton code and tests under this directory. You will need to modify <tt>kvraft/client.go</tt>, <tt>kvraft/server.go</tt>, and perhaps <tt>kvraft/common.go</tt>. (Even if you don’t modify <tt>common.go</tt>, you should submit it as-provided.) For this assignment we give you the option to either use your own implementation of Raft from HW4 or use our solution, which we recommend. Our solution will be provided to you as a binary that you will import in your project.

To get up and running, execute the following commands, as in the previous assignments, and change into the <tt>src/kvraft</tt> directory:

<pre>  # Go needs $GOPATH to be set to the directory containing "src"  $ cd 418/assignment5  $ export GOPATH="$PWD"  $ cd "$GOPATH/src/kvraft"</pre>

To apply the binary, follow these instructions:

<pre>  $ tar -vzxf raft-binary*.tgz  $ rm raft-binary*.tgz  $ mv raft.go src/raft/raft.go  $ cd src/kvraft  $ go test # this should now print "Creating RAFT instance from binary"</pre>

<h2><a id="user-content-part-i" class="anchor" href="https://github.com/theoliao1998/Distributed-Systems/tree/master/5%20Key-Value%20Storage%20Service#part-i" aria-hidden="true"></a>Part I</h2>

The service supports three RPCs: <tt>Put(key, value)</tt>, <tt>Append(key, arg)</tt>, and <tt>Get(key)</tt>. It maintains a simple database of key/value pairs. <tt>Put()</tt> replaces the value for a particular key in the database, <tt>Append(key, arg)</tt> appends arg to key’s value, and <tt>Get()</tt> fetches the current value for a key. An <tt>Append</tt> to a non-existant key should act like <tt>Put</tt>.

You will implement the service as a replicated state machine consisting of several kvservers. Your kvraft client code (<tt>Clerk</tt> in <tt>src/kvraft/client.go</tt>) should try different kvservers it knows about until one responds positively. As long as a client can contact a kvraft server that is a Raft leader in a majority partition, its operations should eventually succeed.

Your kvraft servers should not directly communicate; they should only interact with each other through the Raft log.

Your first task is to implement a solution that works when there are no dropped messages, and no failed servers. Note that your service must provide <em>sequential consistency</em> to applications that use its client interface. That is, completed application calls to the <tt>Clerk.Get()</tt>, <tt>Clerk.Put()</tt>, and <tt>Clerk.Append()</tt> methods in <tt>kvraft/client.go</tt> must appear to have affected all kvservers in the same order, and have at-most-once semantics. A <tt>Clerk.Get(key)</tt> should see the value written by the most recent <tt>Clerk.Put(key, …)</tt> or <tt>Clerk.Append(key, …)</tt> (in the total order).

A reasonable plan of attack may be to first fill in the <tt>Op</tt> struct in <tt>server.go</tt> with the “value” information that kvraft will use Raft to agree on (remember that <tt>Op</tt> field names must start with capital letters, since they will be sent through RPC), and then implement the <tt>PutAppend()</tt> and <tt>Get()</tt> handlers in <tt>server.go</tt>. The handlers should enter an <tt>Op</tt> in the Raft log using <tt>Start()</tt>, and should reply to the client when that log entry is committed. Note that you <strong>cannot</strong> execute an operation until the point at which it is committed in the log (i.e., when it arrives on the Raft <tt>applyCh</tt>).

After calling <tt>Start()</tt>, your kvraft servers will need to wait for Raft to complete agreement. Commands that have been agreed upon arrive on the <tt>applyCh</tt>. You should think carefully about how to arrange your code so that your code will keep reading <tt>applyCh</tt>, while <tt>PutAppend()</tt> and <tt>Get()</tt> handlers submit commands to the Raft log using <tt>Start()</tt>. It is easy to achieve deadlock between the kvserver and its Raft library.

Your solution needs to handle the case in which a leader has called Start() for a client RPC, but loses its leadership before the request is committed to the log. In this case you should arrange for the client to re-send the request to other servers until it finds the new leader. One way to do this is for the server to detect that it has lost leadership, by noticing that a different request has appeared at the index returned by Start(), or that the term reported by Raft.GetState() has changed. If the ex-leader is partitioned by itself, it won’t know about new leaders; but any client in the same partition won’t be able to talk to a new leader either, so it’s OK in this case for the server and client to wait indefinitely until the partition heals. More generally, a kvraft server should not complete a <tt>Get()</tt> RPC if it is not part of a majority.

You have completed Part I when you <strong>reliably</strong> pass the first test in the test suite: “One client”. You may also find that you can pass the “concurrent clients” test, depending on how sophisticated your implementation is. From the <tt>src/kvraft</tt> directory:

<pre>$ go test -v -run Basic=== RUN   TestBasicTest: One client ...  ... Passed--- PASS: TestBasic (15.18s)PASSok  kvraft 15.190s</pre>

<h2><a id="user-content-part-ii" class="anchor" href="https://github.com/theoliao1998/Distributed-Systems/tree/master/5%20Key-Value%20Storage%20Service#part-ii" aria-hidden="true"></a>Part II</h2>

In the face of unreliable connections and node failures, your clients may send RPCs multiple times until it finds a kvraft server that replies positively. One consequence of this is that you must ensure that each application call to <tt>Clerk.Put()</tt> or <tt>Clerk.Append()</tt> must appear in that order just once (i.e., write the key/value database just once).

Thus, your task in Part II is to cope with duplicate client requests, including situations where the client sends a request to a kvraft leader in one term, times out waiting for a reply, and re-sends the request to a new leader in another term. The client request should always execute just once.

You will need to uniquely identify client operations to ensure that they execute just once. You can assume that each clerk has only one outstanding <tt>Put</tt>, <tt>Get</tt>, or <tt>Append</tt>.

For stability, you must make sure that your scheme for duplicate detection frees server memory quickly, for example by having the client tell the servers which RPCs it has heard a reply for. It’s OK to piggyback this information on the next client request.

You have completed Part II when you <strong>reliably</strong> pass all tests through <tt>TestPersistPartitionUnreliable()</tt>.

<pre>$ go test -v=== RUN   TestBasicTest: One client ...  ... Passed--- PASS: TestBasic (15.22s)=== RUN   TestConcurrentTest: concurrent clients ...  ... Passed--- PASS: TestConcurrent (15.83s)=== RUN   TestUnreliableTest: unreliable ...  ... Passed--- PASS: TestUnreliable (16.68s)=== RUN   TestUnreliableOneKeyTest: Concurrent Append to same key, unreliable ...  ... Passed--- PASS: TestUnreliableOneKey (1.40s)=== RUN   TestOnePartitionTest: Progress in majority ...  ... PassedTest: No progress in minority ...  ... PassedTest: Completion after heal ...  ... Passed--- PASS: TestOnePartition (2.54s)=== RUN   TestManyPartitionsOneClientTest: many partitions ...  ... Passed--- PASS: TestManyPartitionsOneClient (24.08s)=== RUN   TestManyPartitionsManyClientsTest: many partitions, many clients ...  ... Passed--- PASS: TestManyPartitionsManyClients (26.12s)=== RUN   TestPersistOneClientTest: persistence with one client ...  ... Passed--- PASS: TestPersistOneClient (18.68s)=== RUN   TestPersistConcurrentTest: persistence with concurrent clients ...  ... Passed--- PASS: TestPersistConcurrent (19.34s)=== RUN   TestPersistConcurrentUnreliableTest: persistence with concurrent clients, unreliable ...  ... Passed--- PASS: TestPersistConcurrentUnreliable (20.37s)=== RUN   TestPersistPartitionTest: persistence with concurrent clients and repartitioning servers...  ... Passed--- PASS: TestPersistPartition (26.91s)=== RUN   TestPersistPartitionUnreliableTest: persistence with concurrent clients and repartitioning servers, unreliable...  ... Passed--- PASS: TestPersistPartitionUnreliable (26.89s)PASSok  kvraft 214.069s</pre>

<h2><a id="user-content-resources-and-advice" class="anchor" href="https://github.com/theoliao1998/Distributed-Systems/tree/master/5%20Key-Value%20Storage%20Service#resources-and-advice" aria-hidden="true"></a>Resources and Advice</h2>

This assignment doesn’t require you to write much code, but you will most likely spend a substantial amount of time thinking and staring at debugging logs to figure out why your implementation doesn’t work. Debugging will be more challenging than in the Raft lab because there are more components that work asynchronously of each other. Start early!

You should implement the service without worrying about the Raft log’s growing without bound. You do not need to implement snapshots (from Section 7 in the paper) to allow garbage collection of old log entries.

As noted, a kvraft server should not complete a <tt>Get()</tt> RPC if it is not part of a majority (so that it does not serve stale data). A simple solution is to enter every <tt>Get()</tt> (as well as each <tt>Put()</tt> and <tt>Append()</tt>) in the Raft log. You don’t have to implement the optimization for read-only operations that is described in Section 8.

In Part I, you should probably modify your client Clerk to remember which server turned out to be the leader for the last RPC, and send the next RPC to that server first. This will avoid wasting time searching for the leader on every RPC.