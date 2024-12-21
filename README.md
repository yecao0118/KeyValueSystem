
# raft-java

# Supported Features
* Leader election
* Log replication
* Snapshot
* Dynamic cluster membership changes

## Quick Start
To deploy a 3-instance Raft cluster locally, execute the following script:<br>
Navigate to the `raft-java-example` directory and run:  
```sh
sh deploy.sh
```
This script will deploy three instances (`example1`, `example2`, `example3`) in the `raft-java-example/env` directory.<br>
It will also create a `client` directory for testing read and write operations on the Raft cluster.<br>
After successful deployment, test write operations using the following command:  
Navigate to `env/client` and run:  
```sh
./bin/run_client.sh "list://127.0.0.1:8051,127.0.0.1:8052,127.0.0.1:8053" hello world
```
For read operations, use:  
```sh
./bin/run_client.sh "list://127.0.0.1:8051,127.0.0.1:8052,127.0.0.1:8053" hello
```

# Usage
The following explains how to use the `raft-java` dependency library to implement a distributed storage system.

## Configuring Dependencies
(Currently not published in the Maven central repository, you need to manually install it locally)
```xml
<dependency>
    <groupId>com.github.raftimpl.raft</groupId>
    <artifactId>raft-java-core</artifactId>
    <version>1.9.0</version>
</dependency>
```

## Defining Data Write and Read Interfaces
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

## Server Usage
1. Implement the `StateMachine` Interface
```java
// This interface includes three main methods for internal Raft calls
public interface StateMachine {
    /**
     * Write snapshot data for the state machine, invoked locally by each node on a schedule.
     * @param snapshotDir Directory to store snapshot data
     */
    void writeSnapshot(String snapshotDir);

    /**
     * Load snapshot data into the state machine, called when a node starts.
     * @param snapshotDir Directory of snapshot data
     */
    void readSnapshot(String snapshotDir);

    /**
     * Apply data to the state machine.
     * @param dataBytes Data in binary format
     */
    void apply(byte[] dataBytes);
}
```

2. Implement the Data Write and Read Interface
```java
// The ExampleService implementation must include the following members:
private RaftNode raftNode;
private ExampleStateMachine stateMachine;
```
```java
// Logic for data writing:
byte[] data = request.toByteArray();
// Synchronously write data to the Raft cluster
boolean success = raftNode.replicate(data, Raft.EntryType.ENTRY_TYPE_DATA);
Example.SetResponse response = Example.SetResponse.newBuilder().setSuccess(success).build();
```
```java
// Logic for data reading, implemented by the specific application state machine
Example.GetResponse response = stateMachine.get(request);
```

3. Server Initialization
```java
// Initialize the RPCServer
RPCServer server = new RPCServer(localServer.getEndPoint().getPort());

// Application-specific state machine
ExampleStateMachine stateMachine = new ExampleStateMachine();

// Configure Raft options, e.g.:
RaftOptions.snapshotMinLogSize = 10 * 1024;
RaftOptions.snapshotPeriodSeconds = 30;
RaftOptions.maxSegmentFileSize = 1024 * 1024;

// Initialize the RaftNode
RaftNode raftNode = new RaftNode(serverList, localServer, stateMachine);

// Register services for inter-node communication
RaftConsensusService raftConsensusService = new RaftConsensusServiceImpl(raftNode);
server.registerService(raftConsensusService);

// Register Raft services for client calls
RaftClientService raftClientService = new RaftClientServiceImpl(raftNode);
server.registerService(raftClientService);

// Register application-provided services
ExampleService exampleService = new ExampleServiceImpl(raftNode, stateMachine);
server.registerService(exampleService);

// Start the RPCServer and initialize the Raft node
server.start();
raftNode.init();
```
