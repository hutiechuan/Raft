raft-java
Raft implementation library for Java.<br>
Inspired by the Raft paper and the open-source implementation by the author of Raft, LogCabin.

Supported Features
Leader election
Log replication
Snapshot
Dynamic changes to cluster members
Quick Start
To deploy a 3-instance Raft cluster on a single local machine, execute the following script:<br>
cd raft-java-example && sh deploy.sh <br>
This script will deploy three instances (example1, example2, example3) in the raft-java-example/env directory;<br>
It also creates a client directory for testing the read and write functions of the Raft cluster.<br>
After successful deployment, test the write operation with the following script:
cd env/client <br>
./bin/run_client.sh "list://127.0.0.1:8051,127.0.0.1:8052,127.0.0.1:8053" hello world <br>
To test the read operation, use the command:<br>
./bin/run_client.sh "list://127.0.0.1:8051,127.0.0.1:8052,127.0.0.1:8053" hello

How to Use
Below is how to use the raft-java dependency library to implement a distributed storage system.

Configuring Dependencies (not yet released to the Maven Central Repository, needs to be manually installed locally)
php
Copy code
<dependency>
    <groupId>com.github.raftimpl.raft</groupId>
    <artifactId>raft-java-core</artifactId>
    <version>1.9.0</version>
</dependency>
Defining Data Write and Read Interfaces
protobuf
Copy code
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
java
Copy code
public interface ExampleService {
    Example.SetResponse set(Example.SetRequest request);
    Example.GetResponse get(Example.GetRequest request);
}
Server-Side Usage
Implement the StateMachine interface class
java
Copy code
// These three methods in this interface are mainly called internally by Raft
public interface StateMachine {
    /**
     * Snapshot the data in the state machine, called locally by each node periodically
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
     * @param dataBytes data binary
     */
    void apply(byte[] dataBytes);
}
Implement the data write and read interfaces
arduino
Copy code
// ExampleService implementation class should include the following members
private RaftNode raftNode;
private ExampleStateMachine stateMachine;
scss
Copy code
// Main logic for data writing
byte[] data = request.toByteArray();
// Synchronously write data to the Raft cluster
boolean success = raftNode.replicate(data, Raft.EntryType.ENTRY_TYPE_DATA);
Example.SetResponse response = Example.SetResponse.newBuilder().setSuccess(success).build();
csharp
Copy code
// Main logic for data reading, implemented by the specific application state machine
Example.GetResponse response = stateMachine.get(request);
Server startup logic
scss
Copy code
// Initialize RPCServer
RPCServer server = new RPCServer(localServer.getEndPoint().getPort());
// Apply the state machine
ExampleStateMachine stateMachine = new ExampleStateMachine();
// Set Raft options, such as:
RaftOptions.snapshotMinLogSize = 10 * 1024;
RaftOptions.snapshotPeriodSeconds = 30;
RaftOptions.maxSegmentFileSize = 1024 * 1024;
// Initialize RaftNode
RaftNode raftNode = new RaftNode(serverList, localServer, stateMachine);
// Register the services for Raft nodes to call each other
RaftConsensusService raftConsensusService = new RaftConsensusServiceImpl(raftNode);
server.registerService(raftConsensusService);
// Register Raft services for clients to call
RaftClientService raftClientService = new RaftClientServiceImpl(raftNode);
server.registerService(raftClientService);
// Register services provided by the application itself
ExampleService exampleService = new ExampleServiceImpl(raftNode, stateMachine);
server.registerService(exampleService);
// Start the RPCServer, initialize Raft nodes
server.start();
raftNode.init();
This translation should provide a clear understanding of how to set up and use the raft-java library.