# raft-java
Raft implementation library for Java.  
Inspired by the [Raft paper](https://github.com/maemual/raft-zh_cn) and Raft author's open-source implementation [LogCabin](https://github.com/logcabin/logcabin).


# Supported Features
- Leader election
- Log replication
- Snapshots
- Dynamic changes to cluster members

## Quick Start
To deploy a 3-instance Raft cluster on a single local machine, execute the following script:  
cd raft-java-example && sh deploy.sh
This script will deploy three instances: example1, example2, example3 in the `raft-java-example/env` directory;  
It also creates a client directory for testing the read/write functionality of the Raft cluster.  
After successful deployment, to test the write operation, run the following script:


## How to Use
Here's how to use the raft-java dependency library to implement a distributed storage system.

## Configuring Dependencies (not yet released to the Maven Central Repository, needs to be manually installed locally)
```xml
<dependency>
    <groupId>com.github.raftimpl.raft</groupId>
    <artifactId>raft-java-core</artifactId>
    <version>1.9.0</version>
</dependency>


## Defining Data Writing and Reading Interfaces

```protobuf
message SetRequest {
    string key = 1;
    string value = 2;
}
message SetResponse {
    bool success = 1;
}
message GetRequest {
    string key = 1;
}
message GetResponse {
    string value = 1;
}
```
```java
public interface ExampleService {
    Example.SetResponse set(Example.SetRequest request);
    Example.GetResponse get(Example.GetRequest request);
}
```

## Server-Side Usage
1. Implement the StateMachine interface implementation class
```java

// These three methods in this interface are primarily called internally by Raft
public interface StateMachine {
    /**
     * Perform a snapshot of the data in the state machine, called locally by each node periodically
     * @param snapshotDir snapshot data output directory
     */
    void writeSnapshot(String snapshotDir);

    /**
     * Read snapshot into the state machine, called when the node starts
     * @param snapshotDir snapshot data directory
     */
    void readSnapshot(String snapshotDir);

    /**
     * Apply data to the state machine
     * @param dataBytes binary data
     */
    void apply(byte[] dataBytes);
}

```

2. Implement the data writing and reading interfaces
```
// The ExampleService implementation class should include the following members
private RaftNode raftNode;
private ExampleStateMachine stateMachine;

// Main logic for data writing
byte[] data = request.toByteArray();
// Synchronously write data into the Raft cluster
boolean success = raftNode.replicate(data, Raft.EntryType.ENTRY_TYPE_DATA);
Example.SetResponse response = Example.SetResponse.newBuilder().setSuccess(success).build();

// Main logic for data reading, implemented by the specific application state machine
Example.GetResponse response = stateMachine.get(request);

```

3. Server startup logic
```
/// Initialize the RPCServer
RPCServer server = new RPCServer(localServer.getEndPoint().getPort());

// Apply the state machine
ExampleStateMachine stateMachine = new ExampleStateMachine();

// Set Raft options, such as:
RaftOptions.snapshotMinLogSize = 10 * 1024;
RaftOptions.snapshotPeriodSeconds = 30;
RaftOptions.maxSegmentFileSize = 1024 * 1024;

// Initialize the RaftNode
RaftNode raftNode = new RaftNode(serverList, localServer, stateMachine);

// Register the services for Raft nodes to call each other
RaftConsensusService raftConsensusService = new RaftConsensusServiceImpl(raftNode);
server.registerService(raftConsensusService);

// Register the Raft services for clients to call
RaftClientService raftClientService = new RaftClientServiceImpl(raftNode);
server.registerService(raftClientService);

// Register the services provided by the application itself
ExampleService exampleService = new ExampleServiceImpl(raftNode, stateMachine);
server.registerService(exampleService);

// Start the RPCServer, initialize the Raft nodes
server.start();
raftNode.init();
```