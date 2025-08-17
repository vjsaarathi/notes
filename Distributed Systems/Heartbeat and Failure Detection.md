---
tags:
  - distributed-systems
  - heartbeat
  - failure-detection
  - monitoring
  - hdfs
  - reliability
  - network
---

## The Fundamental Problem

In a distributed system with thousands of machines, failures happen constantly. The system must quickly detect when machines fail so it can:
1. Stop sending new requests to failed machines
2. Re-replicate data that was stored on failed machines
3. Update metadata to reflect current system state

But how do you detect failure across a network where messages can be delayed, lost, or machines can be temporarily unreachable?

## Failure Detection Challenges

### What Does "Failure" Mean?
```
Scenario 1: Machine completely powered off
→ Clear failure - no response to any requests

Scenario 2: Machine running but network cable unplugged  
→ Machine is fine, but unreachable - is this failure?

Scenario 3: Machine extremely overloaded, responding slowly
→ Technically working, but practically unusable

Scenario 4: Network partition - machine reachable from some nodes but not others
→ Partial failure - different views of system state
```

### The Impossibility of Perfect Detection
In distributed systems, you cannot perfectly distinguish between:
- Machine failure
- Network delay
- Network partition
- Extreme slowness

This leads to fundamental tradeoffs in failure detection systems.

## Heartbeat Mechanism

### Basic Concept
```python
# Each machine sends periodic "I'm alive" messages
def send_heartbeat():
    while True:
        message = {
            'node_id': 'machine-47',
            'timestamp': current_time(),
            'status': 'alive'
        }
        send_to_coordinator(message)
        sleep(heartbeat_interval)
```

### Why Machines Ping the Registry (Not Vice Versa)

#### Option A: Registry Pings Machines (Centralized Monitoring)
```python
def monitor_cluster():
    for machine in all_machines:
        response = ping(machine)
        if not response:
            mark_as_failed(machine)
```

**Problems**:
- Registry becomes bottleneck (10,000 machines × ping frequency)
- Registry spends all time monitoring instead of serving clients
- Single point of monitoring failure

#### Option B: Machines Ping Registry (Distributed Monitoring)
```python
def machine_heartbeat_loop():
    while True:
        send_heartbeat_to_registry()
        sleep(3_seconds)
```

**Advantages**:
- Registry load is distributed across time
- Registry focuses on serving client requests  
- Each machine manages its own monitoring overhead
- Natural load balancing (machines send at different times)

## HDFS Heartbeat Implementation

### DataNode Heartbeat
```python
def datanode_heartbeat():
    while True:
        heartbeat = {
            'datanode_id': self.node_id,
            'timestamp': current_time(),
            'capacity': self.total_space,
            'used': self.used_space,
            'remaining': self.free_space,
            'block_pool_used': self.hdfs_used_space,
            'blocks': self.get_block_list(),
            'failed_volumes': self.get_failed_disks()
        }
        
        response = namenode.send_heartbeat(heartbeat)
        
        # Process commands from NameNode
        self.process_commands(response.commands)
        
        sleep(3_seconds)  # Default heartbeat interval
```

### NameNode Heartbeat Processing
```python
def process_heartbeat(heartbeat):
    node_id = heartbeat.datanode_id
    
    # Update last seen time
    self.datanodes[node_id].last_heartbeat = current_time()
    
    # Update capacity information
    self.datanodes[node_id].update_capacity(heartbeat)
    
    # Update block locations
    self.update_block_locations(node_id, heartbeat.blocks)
    
    # Generate response commands
    commands = []
    
    # Block replication commands
    under_replicated = self.find_under_replicated_blocks()
    for block in under_replicated:
        if self.should_replicate_on_node(block, node_id):
            commands.append(ReplicateCommand(block, target_nodes))
    
    # Block deletion commands  
    over_replicated = self.find_over_replicated_blocks()
    for block in over_replicated:
        if node_id in block.replicas:
            commands.append(DeleteCommand(block))
    
    return HeartbeatResponse(commands)
```

### Failure Detection Parameters
```python
HEARTBEAT_INTERVAL = 3_seconds        # How often DataNodes send heartbeats
HEARTBEAT_TIMEOUT = 10 * 60_seconds   # 10 minutes without heartbeat = dead
RECHECK_INTERVAL = 5 * 60_seconds     # How often to check for dead nodes
```

**Why these values?**
- **3 seconds**: Fast enough to detect problems quickly, not so fast as to spam network
- **10 minutes**: Allows for temporary network issues, GC pauses, etc.
- **5 minutes recheck**: Balance between responsiveness and computational overhead

## Handling False Positives

### The Problem
```
Timeline:
10:00 - DataNode sends heartbeat
10:01 - Network becomes congested
10:02 - NameNode doesn't receive heartbeat (network delay)
10:03 - NameNode marks DataNode as "suspicious"
10:04 - Heartbeat finally arrives (3-second delay)
10:05 - DataNode marked as healthy again
```

False positives cause:
- Unnecessary re-replication (wastes network, storage)
- Metadata churn (constant updates)
- Resource waste (CPU cycles on recovery)

### Mitigation Strategies

#### Grace Periods
```python
def check_node_health():
    for node in all_datanodes:
        time_since_heartbeat = current_time() - node.last_heartbeat
        
        if time_since_heartbeat > HEARTBEAT_TIMEOUT:
            if node.status != 'DEAD':
                node.status = 'STALE'  # Grace period
                node.stale_since = current_time()
        
        elif time_since_heartbeat > (HEARTBEAT_TIMEOUT + GRACE_PERIOD):
            node.status = 'DEAD'
            self.initiate_recovery(node)
```

#### Exponential Backoff
```python
def adaptive_heartbeat():
    base_interval = 3_seconds
    current_interval = base_interval
    
    while True:
        try:
            send_heartbeat()
            current_interval = base_interval  # Reset on success
        except NetworkError:
            current_interval = min(current_interval * 2, 60_seconds)  # Cap at 1 minute
        
        sleep(current_interval)
```

## Advanced Failure Detection

### Split-Brain Prevention
**Problem**: Network partition causes nodes to have different views of cluster state

```
Partition A: NameNode + 40% of DataNodes
Partition B: 60% of DataNodes (can't reach NameNode)

Both sides think the other side has failed!
```

**Solution**: Quorum-based decisions
```python
def is_cluster_healthy():
    total_expected_nodes = len(self.all_registered_datanodes)
    active_nodes = len(self.recently_active_datanodes)
    
    # Require majority to make decisions
    return active_nodes > (total_expected_nodes / 2)
```

### Cascading Failure Detection
**Problem**: Legitimate high load can trigger false failure detection

```
Scenario:
1. Cluster becomes very busy processing large job
2. All nodes become slow responding to heartbeats
3. NameNode marks many nodes as failed
4. Remaining nodes get even more load
5. More nodes marked as failed (cascade effect)
```

**Mitigation**:
```python
def adaptive_timeout():
    recent_response_times = self.get_recent_heartbeat_times()
    avg_response_time = statistics.mean(recent_response_times)
    
    # Adjust timeout based on current cluster performance
    if avg_response_time > normal_threshold:
        timeout = HEARTBEAT_TIMEOUT * 2  # Be more lenient during high load
    else:
        timeout = HEARTBEAT_TIMEOUT
    
    return timeout
```

### Multi-Level Health Monitoring
```python
class NodeHealthMonitor:
    def assess_node_health(self, node):
        health_score = 0
        
        # Heartbeat responsiveness (40% weight)
        if node.avg_heartbeat_delay < 1_second:
            health_score += 40
        elif node.avg_heartbeat_delay < 5_seconds:
            health_score += 20
        
        # Disk health (30% weight)  
        if node.failed_disks == 0:
            health_score += 30
        elif node.failed_disks < 2:
            health_score += 15
        
        # Network performance (20% weight)
        if node.avg_network_latency < 10_ms:
            health_score += 20
        elif node.avg_network_latency < 50_ms:
            health_score += 10
            
        # CPU/Memory utilization (10% weight)
        if node.cpu_usage < 80% and node.memory_usage < 90%:
            health_score += 10
        
        return health_score  # 0-100 scale
```

## Recovery Operations

### Block Re-replication Process
```python
def handle_node_failure(failed_node):
    # 1. Mark all blocks on failed node as under-replicated
    affected_blocks = self.get_blocks_on_node(failed_node)
    
    # 2. Prioritize critical blocks (those with only 1 remaining replica)
    critical_blocks = [b for b in affected_blocks if b.replica_count == 1]
    important_blocks = [b for b in affected_blocks if b.replica_count == 2]
    
    # 3. Schedule re-replication (critical blocks first)
    replication_queue = critical_blocks + important_blocks
    
    for block in replication_queue:
        source_node = self.choose_replication_source(block)
        target_node = self.choose_replication_target(block)
        
        self.schedule_replication(block, source_node, target_node)
```

### Choosing Replication Targets
```python
def choose_replication_target(self, block):
    candidates = []
    
    for node in self.healthy_datanodes:
        if node.has_space_for_block(block):
            score = 0
            
            # Prefer nodes with more free space
            score += node.free_space_ratio * 40
            
            # Prefer nodes with lower current load
            score += (100 - node.current_load) * 30
            
            # Prefer different rack from existing replicas
            if node.rack not in block.existing_racks:
                score += 20
                
            # Prefer closer network distance
            score += (100 - node.network_distance) * 10
            
            candidates.append((node, score))
    
    # Return highest scoring candidate
    return max(candidates, key=lambda x: x[1])[0]
```

## Monitoring and Alerting

### Health Metrics to Track
```python
class ClusterHealthMetrics:
    def collect_metrics(self):
        return {
            # Node-level metrics
            'total_nodes': len(self.all_datanodes),
            'healthy_nodes': len(self.healthy_datanodes),
            'stale_nodes': len(self.stale_datanodes),
            'dead_nodes': len(self.dead_datanodes),
            
            # Block-level metrics
            'total_blocks': len(self.all_blocks),
            'under_replicated_blocks': len(self.under_replicated_blocks),
            'over_replicated_blocks': len(self.over_replicated_blocks),
            'corrupt_blocks': len(self.corrupt_blocks),
            
            # Performance metrics
            'avg_heartbeat_processing_time': self.avg_heartbeat_time,
            'pending_replication_requests': len(self.replication_queue),
            'cluster_capacity_used': self.total_used / self.total_capacity
        }
```

### Alert Conditions
```python
def check_alert_conditions(self, metrics):
    alerts = []
    
    if metrics.dead_nodes > metrics.total_nodes * 0.1:  # 10% nodes dead
        alerts.append('CRITICAL: High node failure rate')
    
    if metrics.under_replicated_blocks > 1000:
        alerts.append('WARNING: Many under-replicated blocks')
    
    if metrics.cluster_capacity_used > 0.85:  # 85% full
        alerts.append('WARNING: Cluster approaching capacity')
    
    if metrics.avg_heartbeat_processing_time > 100_ms:
        alerts.append('WARNING: Slow heartbeat processing')
    
    return alerts
```

## Connection to Other Concepts

- [[Metadata Management]] - How failure detection updates metadata
- [[Fault Tolerance and Replication]] - What happens after failure detection
- [[HDFS Architecture]] - How heartbeat fits into overall system
- [[Performance vs Reliability Tradeoffs]] - Tuning heartbeat parameters

---

*Previous: [[Metadata Management]] | Next: [[HDFS Architecture]]*