Quick Start

Deploy a 3-instance cluster locally:
cd distribute-java-cluster && sh deploy.sh

This script will deploy three instances (example1, example2, example3) in distribute-java-cluster/env. It will also create a client directory for testing Raft cluster read/write functionality.

Test Write Operation:
cd env/client
./bin/run_client.sh "list://127.0.0.1:8051,127.0.0.1:8052,127.0.0.1:8053" hello world

Test Read Operation:
./bin/run_client.sh "list://127.0.0.1:8051,127.0.0.1:8052,127.0.0.1:8053" hello

Usage

Define Data Write/Read Interfaces:
protobuf
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
public interface ExampleService {
Example.SetResponse set(Example.SetRequest request);
Example.GetResponse get(Example.GetRequest request);
}

Server Usage

Implement the StateMachine interface:

public interface StateMachine {
/**
* Write snapshot of state machine data.
* Called periodically on each node.
* @param snapshotDir snapshot output directory
*/
void writeSnap(String snapshotDir);

/**  
 * Load snapshot into the state machine.  
 * Called when a node starts.  
 * @param snapshotDir snapshot data directory  
 */  
void readSnap(String snapshotDir);  

/**  
 * Apply data to the state machine.  
 * @param dataBytes binary data  
 */  
void applyData(byte[] dataBytes);  


}

Implement Data Write and Read Logic:

// Members inside ExampleService implementation
private RaftNode raftNode;
private ExampleStateMachine stateMachine;

// Write data logic
byte[] data = request.toByteArray();
boolean success = raftNode.replicate(data, Raft.EntryType.ENTRY_TYPE_DATA);
Example.SetResponse response = Example.SetResponse.newBuilder()
.setSuccess(success)
.build();

// Read data logic
Example.GetResponse response = stateMachine.get(request);

Server Startup:

// Initialize RPCServer
RPCServer server = new RPCServer(localServer.getEndPoint().getPort());

// Application state machine
ExampleStateMachine stateMachine = new ExampleStateMachine();

// Configure Raft options
RaftOptions.snapshotMinLogSize = 10 * 1024;
RaftOptions.snapshotPeriodSeconds = 30;
RaftOptions.maxSegmentFileSize = 1024 * 1024;

// Initialize RaftNode
RaftNode raftNode = new RaftNode(serverList, localServer, stateMachine);

// Register Raft consensus service
RaftConsensusService raftConsensusService = new RaftConsensusServiceImpl(raftNode);
server.registerService(raftConsensusService);

// Register Raft client service
RaftClientService raftClientService = new RaftClientServiceImpl(raftNode);
server.registerService(raftClientService);

// Register application service
ExampleService exampleService = new ExampleServiceImpl(raftNode, stateMachine);
server.registerService(exampleService);

// Start services
server.start();
raftNode.init();