# Distributed Locks

## What are Distributed Locks?

Distributed locks are synchronization primitives that coordinate access to shared resources across multiple nodes or processes in a distributed system, ensuring that only one process can access a critical resource at any given time. Think of distributed locks like the reservation system for a single bathroom in a busy office building - multiple people might want to use it simultaneously, but the reservation system ensures that only one person has access at a time while others wait their turn. Without this coordination, you'd have conflicts and an unusable situation.

In distributed computing, multiple services or instances often need to access shared resources like databases, files, or external APIs that can only handle one operation at a time. Traditional thread-level locks that work within a single process are insufficient when coordination is needed across multiple servers or services. Distributed locks provide a way for processes running on different machines to coordinate access to these shared resources safely.

## Why Distributed Locks are Necessary

### Preventing Race Conditions

Race conditions occur when multiple processes attempt to modify the same resource simultaneously, leading to inconsistent or corrupted results. In distributed systems, these race conditions can have serious consequences because the processes are running on different machines and can't use traditional synchronization mechanisms.

Consider a scenario where multiple instances of a service need to process unique customer IDs. Without coordination, two instances might generate the same customer ID simultaneously, causing data integrity issues. Distributed locks prevent this by ensuring only one instance can generate IDs at a time.

### Ensuring Single Instance Operations

Many operations in distributed systems should only be performed by one instance at a time, even when multiple instances of a service are running. Examples include scheduled batch jobs, database migrations, cache warming operations, or sending critical notifications. Distributed locks ensure that these operations don't run concurrently across different instances.

### Resource Contention Management

Some shared resources, like external APIs with strict rate limits or legacy systems that can't handle concurrent access, require careful coordination. Distributed locks provide a way to serialize access to these resources, preventing overload and ensuring fair usage across multiple consuming services.

### Maintaining Data Consistency

In systems where multiple processes might update related data, distributed locks help maintain consistency by ensuring that related operations complete atomically from the perspective of the distributed system. While this doesn't provide ACID transaction guarantees across services, it does provide a higher level of coordination than uncoordinated access.

## Types of Distributed Lock Implementations

### Database-Based Locks

Database-based distributed locks use a shared database table to coordinate access across processes. A process acquires a lock by inserting or updating a record in a lock table, and releases the lock by deleting or updating the record. This approach leverages the database's transaction capabilities to ensure atomicity of lock operations.

Database locks are reliable and easy to implement, especially in systems that already depend on a shared database. However, they can create performance bottlenecks if many processes are competing for locks, and they introduce a dependency on database availability for lock operations.

### Redis-Based Locks

Redis-based locks use the atomic operations provided by Redis to implement distributed coordination. The most common approach uses the `SET` command with the `NX` (not exists) and `EX` (expiration) options to atomically set a key only if it doesn't already exist, with an automatic expiration time.

Redis locks are fast and efficient, with lower latency than database-based approaches. They also provide natural expiration mechanisms that help prevent deadlocks. However, they require additional infrastructure (Redis) and careful consideration of failure scenarios to ensure reliability.

### Consensus-Based Locks

Consensus-based locks use distributed consensus algorithms like Raft or Paxos to coordinate lock operations across multiple nodes. Systems like Apache ZooKeeper, etcd, or Consul provide distributed locking capabilities built on proven consensus algorithms.

These locks provide strong consistency guarantees and are highly reliable, even in the face of network partitions and node failures. However, they have higher implementation complexity and operational overhead compared to simpler approaches.

### Cloud Provider Locks

Cloud providers offer managed distributed locking services that handle the complexity of implementation and operation. Examples include AWS DynamoDB's conditional writes, Google Cloud Firestore transactions, or Azure Cosmos DB's optimistic concurrency control.

These services offload the operational burden but may have limitations in terms of performance characteristics or vendor lock-in considerations.

## Distributed Lock Challenges

### Lock Expiration and Timeouts

One of the biggest challenges in distributed locking is handling lock expiration. Locks must have timeouts to prevent deadlocks when lock holders crash or become unreachable. However, determining appropriate timeout values is difficult - too short and locks may expire while legitimate work is still in progress, too long and failed processes may hold locks for extended periods.

The challenge is compounded by network delays and clock skew between different machines. A process might think it still holds a lock while the lock has actually expired from the perspective of other processes.

### Split-Brain Scenarios

Network partitions can create situations where different parts of the system have different views of lock ownership. A process might believe it still holds a lock while another process has acquired the same lock due to network connectivity issues. This can lead to multiple processes believing they have exclusive access to a resource.

### Performance and Scalability

Distributed locks can become performance bottlenecks if not designed carefully. High contention for popular locks can create queuing effects that reduce overall system throughput. Additionally, the overhead of acquiring and releasing locks adds latency to operations.

### Deadlock Prevention

Deadlocks can occur when multiple processes wait for locks held by each other in a circular dependency. In distributed systems, detecting and resolving deadlocks is more complex than in single-process systems because the full system state isn't visible to any single process.

## Real-World Applications

### Scheduled Job Coordination

In microservices architectures where multiple instances of a service run for high availability, scheduled jobs like data backups, report generation, or maintenance tasks should only run on one instance at a time. Distributed locks ensure that these jobs don't run concurrently across multiple instances.

For example, a daily report generation job might acquire a distributed lock before starting processing. If multiple instances of the service are running, only one will successfully acquire the lock and perform the work, while others skip the execution or wait for the next scheduled time.

### Database Migration Coordination

Database schema migrations in distributed systems require careful coordination to ensure they only run once, even when multiple application instances are deployed simultaneously. Distributed locks prevent race conditions where multiple instances might attempt to run the same migration concurrently.

### Critical Resource Access

Some resources, like external payment gateways or legacy APIs, have strict concurrency limitations. Distributed locks provide a way to coordinate access to these resources across multiple services, preventing overload and ensuring fair usage.

For instance, an integration with a legacy mainframe system that can only handle one connection at a time would use distributed locks to coordinate access across multiple microservices that need to interact with the system.

### Unique Identifier Generation

Systems that need to generate unique identifiers (like order numbers or transaction IDs) across multiple instances can use distributed locks to coordinate the generation process. This ensures that no duplicate identifiers are created even when multiple services are generating IDs simultaneously.

### Cache Warming and Data Loading

Cache warming operations or bulk data loading processes often should only be performed by one instance at a time to avoid overwhelming downstream systems or creating duplicate work. Distributed locks coordinate these operations across multiple instances.

## Simple Distributed Lock Implementation

```python
import time
import uuid
import redis
from typing import Optional, Union
from contextlib import contextmanager

class DistributedLock:
    def __init__(self, redis_client: redis.Redis, lock_name: str, 
                 timeout: int = 10, retry_delay: float = 0.1):
        self.redis = redis_client
        self.lock_name = f"lock:{lock_name}"
        self.timeout = timeout
        self.retry_delay = retry_delay
        self.identifier = None
    
    def acquire(self, blocking: bool = True, timeout: Optional[int] = None) -> bool:
        """Acquire the distributed lock"""
        identifier = str(uuid.uuid4())
        lock_timeout = timeout or self.timeout
        end_time = time.time() + lock_timeout if blocking else time.time()
        
        while time.time() < end_time:
            # Try to acquire the lock with SET NX EX
            if self.redis.set(self.lock_name, identifier, nx=True, ex=self.timeout):
                self.identifier = identifier
                return True
            
            if not blocking:
                return False
            
            time.sleep(self.retry_delay)
        
        return False
    
    def release(self) -> bool:
        """Release the distributed lock"""
        if not self.identifier:
            return False
        
        # Use Lua script to ensure atomic check-and-delete
        lua_script = """
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
        else
            return 0
        end
        """
        
        result = self.redis.eval(lua_script, 1, self.lock_name, self.identifier)
        if result:
            self.identifier = None
            return True
        return False
    
    def extend(self, additional_time: int) -> bool:
        """Extend the lock expiration time"""
        if not self.identifier:
            return False
        
        lua_script = """
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("expire", KEYS[1], ARGV[2])
        else
            return 0
        end
        """
        
        result = self.redis.eval(lua_script, 1, self.lock_name, 
                               self.identifier, additional_time)
        return bool(result)
    
    def is_locked(self) -> bool:
        """Check if the lock is currently held"""
        return self.redis.exists(self.lock_name)
    
    def __enter__(self):
        """Context manager entry"""
        if not self.acquire():
            raise RuntimeError(f"Could not acquire lock: {self.lock_name}")
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        """Context manager exit"""
        self.release()

class DistributedLockManager:
    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
    
    def lock(self, name: str, timeout: int = 10, retry_delay: float = 0.1) -> DistributedLock:
        """Create a new distributed lock"""
        return DistributedLock(self.redis, name, timeout, retry_delay)
    
    @contextmanager
    def acquire_lock(self, name: str, timeout: int = 10, blocking: bool = True):
        """Context manager for acquiring locks"""
        lock = self.lock(name, timeout)
        try:
            if lock.acquire(blocking=blocking, timeout=timeout):
                yield lock
            else:
                raise RuntimeError(f"Could not acquire lock: {name}")
        finally:
            lock.release()

# Example usage and demonstration
def demo_distributed_locks():
    # Connect to Redis
    redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)
    lock_manager = DistributedLockManager(redis_client)
    
    print("=== Basic Lock Usage ===")
    
    # Basic lock acquisition and release
    lock = lock_manager.lock("demo_resource")
    if lock.acquire():
        print("Lock acquired successfully")
        time.sleep(2)  # Simulate work
        lock.release()
        print("Lock released")
    else:
        print("Failed to acquire lock")
    
    print("\n=== Context Manager Usage ===")
    
    # Using context manager
    try:
        with lock_manager.acquire_lock("demo_resource", timeout=5):
            print("Working with locked resource...")
            time.sleep(1)
            print("Work completed")
    except RuntimeError as e:
        print(f"Lock acquisition failed: {e}")
    
    print("\n=== Concurrent Access Demo ===")
    
    # Simulate concurrent access
    import threading
    
    def worker(worker_id: int, work_duration: int):
        try:
            with lock_manager.acquire_lock("shared_resource", timeout=10):
                print(f"Worker {worker_id} acquired lock")
                time.sleep(work_duration)
                print(f"Worker {worker_id} completed work")
        except RuntimeError:
            print(f"Worker {worker_id} failed to acquire lock")
    
    # Start multiple workers
    threads = []
    for i in range(3):
        thread = threading.Thread(target=worker, args=(i, 2))
        threads.append(thread)
        thread.start()
    
    # Wait for all workers to complete
    for thread in threads:
        thread.join()
    
    print("\n=== Lock Extension Demo ===")
    
    # Demonstrate lock extension
    lock = lock_manager.lock("extendable_resource", timeout=5)
    if lock.acquire():
        print("Lock acquired with 5 second timeout")
        time.sleep(3)
        
        if lock.extend(10):
            print("Lock extended by 10 seconds")
            time.sleep(3)
            print("Additional work completed")
        else:
            print("Failed to extend lock")
        
        lock.release()
        print("Lock released")

# Example: Scheduled job with distributed lock
class ScheduledJobWithLock:
    def __init__(self, redis_client: redis.Redis, job_name: str):
        self.lock_manager = DistributedLockManager(redis_client)
        self.job_name = job_name
        self.lock_name = f"scheduled_job:{job_name}"
    
    def run_job(self):
        """Run job with distributed lock to prevent concurrent execution"""
        try:
            with self.lock_manager.acquire_lock(self.lock_name, timeout=300, blocking=False):
                print(f"Starting scheduled job: {self.job_name}")
                self._do_work()
                print(f"Completed scheduled job: {self.job_name}")
        except RuntimeError:
            print(f"Job {self.job_name} is already running on another instance - skipping")
    
    def _do_work(self):
        """Simulate actual job work"""
        print("Processing data...")
        time.sleep(2)
        print("Generating report...")
        time.sleep(1)
        print("Sending notifications...")
        time.sleep(1)

if __name__ == "__main__":
    # Run the demonstration
    demo_distributed_locks()
    
    print("\n=== Scheduled Job Example ===")
    
    # Example of scheduled job with locking
    redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)
    job = ScheduledJobWithLock(redis_client, "daily_report")
    
    # Simulate multiple instances trying to run the same job
    import threading
    
    def run_job_instance(instance_id: int):
        print(f"Instance {instance_id} attempting to run job")
        job.run_job()
    
    # Start multiple job instances simultaneously
    job_threads = []
    for i in range(3):
        thread = threading.Thread(target=run_job_instance, args=(i,))
        job_threads.append(thread)
        thread.start()
    
    # Wait for all instances to complete
    for thread in job_threads:
        thread.join()
    
    print("All job instances completed")
```

## Common Interview Questions

**Q: What are distributed locks and when would you use them?**

Distributed locks are synchronization primitives that coordinate access to shared resources across multiple processes or nodes in a distributed system. Use them when you need to ensure only one process accesses a resource at a time across multiple machines, such as: preventing duplicate scheduled jobs across service instances, coordinating database migrations, serializing access to rate-limited APIs, generating unique identifiers across services, or ensuring single-instance operations like cache warming. They're essential when traditional thread-level locks aren't sufficient due to processes running on different machines.

**Q: What are the main challenges with implementing distributed locks?**

Key challenges include: lock expiration timing (balancing deadlock prevention with legitimate long-running operations), split-brain scenarios (network partitions causing multiple processes to think they hold the same lock), performance bottlenecks (high contention can reduce throughput), clock synchronization (time differences between nodes affect timeouts), failure detection (determining when a lock holder has crashed), and deadlock prevention (circular dependencies are harder to detect in distributed systems). The fundamental challenge is achieving coordination across unreliable networks with independent clocks.

**Q: What are different approaches to implementing distributed locks?**

Main approaches include: Database-based (using table rows with transactions, reliable but potentially slow), Redis-based (using atomic SET NX EX commands, fast but requires additional infrastructure), Consensus-based (ZooKeeper, etcd, Consul using Raft/Paxos, highly reliable but complex), and Cloud provider services (DynamoDB conditional writes, Firestore transactions, managed but potentially vendor-specific). Choose based on existing infrastructure, consistency requirements, performance needs, and operational complexity tolerance.

**Q: How do you handle lock expiration and prevent deadlocks in distributed systems?**

Handle expiration through: setting appropriate timeouts based on expected operation duration, implementing lock extension mechanisms for long-running operations, using heartbeat systems to detect failed processes, and implementing graceful degradation when locks expire unexpectedly. Prevent deadlocks through: avoiding nested lock acquisition, using consistent lock ordering when multiple locks are needed, implementing timeout-based deadlock detection, using try-lock patterns instead of blocking indefinitely, and designing operations to be idempotent when possible. The key is balancing safety (preventing resource conflicts) with liveness (avoiding permanent blocking).

## Distributed Lock Best Practices

### Choose Appropriate Timeout Values

Set lock timeouts based on realistic estimates of operation duration plus safety margins. Monitor actual operation times and adjust timeouts accordingly. Consider implementing dynamic timeouts based on historical performance data for different types of operations.

### Implement Lock Extension for Long Operations

For operations that might take longer than expected, implement lock extension mechanisms that allow processes to renew their locks before expiration. This prevents legitimate operations from losing locks due to conservative timeout settings.

### Use Unique Identifiers for Lock Ownership

Always use unique identifiers (like UUIDs) to identify lock owners and verify ownership before releasing locks. This prevents processes from accidentally releasing locks held by other processes and helps detect potential bugs in lock management logic.

### Design for Lock Failure Scenarios

Plan for situations where locks can't be acquired or are lost unexpectedly. Implement graceful degradation, idempotent operations where possible, and proper error handling. Consider whether operations should fail fast or retry when locks aren't available.

### Monitor Lock Usage and Performance

Implement comprehensive monitoring for lock acquisition times, hold durations, contention levels, and failure rates. This data helps identify performance bottlenecks, tune timeout values, and detect problematic usage patterns.

### Keep Critical Sections Small

Minimize the amount of work done while holding locks to reduce contention and improve overall system throughput. Prepare data and perform non-critical operations outside of locked sections when possible.

Distributed locks are a powerful tool for coordination in distributed systems, but they require careful design and implementation to avoid becoming performance bottlenecks or sources of system instability. When used appropriately, they enable safe resource sharing and coordination across distributed processes.
