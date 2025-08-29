# Circuit Breaker Pattern

## What is the Circuit Breaker Pattern?

The Circuit Breaker pattern is a design pattern used to detect failures and prevent cascading system failures by temporarily stopping calls to a failing service. Think of it like the electrical circuit breaker in your home - when an electrical fault occurs that could damage your appliances or cause a fire, the circuit breaker immediately cuts the power to protect the entire electrical system. Similarly, when a software service starts failing, a circuit breaker stops sending requests to that service to prevent the failure from spreading throughout your distributed system.

In distributed systems, services often depend on other services, creating chains of dependencies. When one service in the chain fails, it can cause a cascading failure that brings down the entire system. The circuit breaker pattern acts as a protective mechanism that monitors service calls and "trips" (opens) when failures exceed a threshold, preventing further damage and giving the failing service time to recover.

## How Circuit Breakers Work

### The Three States

A circuit breaker operates in three distinct states, each serving a specific purpose in the failure detection and recovery process.

**Closed State** represents normal operation where all requests are allowed to pass through to the target service. The circuit breaker monitors the success and failure rates of these requests, keeping track of recent results to determine if the service is healthy. In this state, the system operates normally with full functionality, but the circuit breaker is continuously watching for signs of trouble.

**Open State** occurs when the failure rate exceeds the configured threshold. In this state, the circuit breaker immediately rejects all requests without attempting to call the failing service, returning a predefined error response or fallback result. This prevents the system from wasting resources on calls that are likely to fail and protects the failing service from additional load that might prevent its recovery.

**Half-Open State** is a trial period that begins after a specified timeout when the circuit breaker is in the open state. During this phase, the circuit breaker allows a limited number of test requests to pass through to check if the service has recovered. If these test requests succeed, the circuit breaker transitions back to the closed state. If they fail, it returns to the open state for another timeout period.

### Failure Detection and Thresholds

Circuit breakers use various metrics to detect failures and decide when to trip. Common failure indicators include response timeouts, HTTP error status codes (5xx), connection failures, and application-specific exceptions. The breaker typically maintains a sliding window of recent requests to calculate failure rates rather than looking at absolute numbers of failures.

Threshold configuration is crucial for effective circuit breaker operation. Setting thresholds too low can cause unnecessary tripping during minor issues, while setting them too high might allow cascading failures to occur. Most implementations allow configuration of failure percentage thresholds (e.g., trip when 50% of requests fail), minimum request volumes (e.g., only consider failure rates after at least 10 requests), and timeout periods for state transitions.

### Fallback Mechanisms

When a circuit breaker is open and rejecting requests, it should provide meaningful fallback responses rather than simply returning errors. Fallback strategies might include returning cached data, default values, simplified responses, or gracefully degraded functionality. The goal is to maintain system functionality, even if at a reduced level, rather than complete service failure.

## Why Circuit Breakers are Essential

### Preventing Cascading Failures

In distributed systems, one slow or failing service can trigger a cascade of failures throughout the system. When Service A calls Service B, and Service B is slow to respond, Service A's threads or connection pools can become exhausted waiting for responses. This resource exhaustion can cause Service A to become unresponsive, which then affects any services that depend on Service A.

Circuit breakers prevent this cascade by quickly failing fast when a service is having problems, freeing up resources that would otherwise be tied up in failed requests. This allows the rest of the system to continue operating normally while the problematic service recovers.

### Resource Protection

Failed service calls consume valuable system resources including threads, memory, network connections, and database connections. When a service is down, continuing to send requests wastes these resources and can lead to resource exhaustion in the calling service. Circuit breakers protect these resources by stopping calls to failed services immediately.

This resource protection is particularly important in high-traffic systems where resource exhaustion can quickly spread from one component to many, creating system-wide outages that are difficult to recover from.

### Improved System Resilience

Circuit breakers improve overall system resilience by isolating failures and providing controlled degradation. Instead of complete system failure, circuit breakers enable systems to continue operating with reduced functionality, providing better user experience during outages and maintaining critical business operations.

### Faster Recovery

By stopping traffic to failing services, circuit breakers give those services an opportunity to recover without being overwhelmed by incoming requests. This can significantly reduce recovery time, as the failing service doesn't need to handle the additional load while trying to resolve its issues.

## Real-World Applications

### E-commerce Platform

Consider an e-commerce platform where the main application depends on several microservices: user service, product catalog, inventory service, payment service, and recommendation engine. During Black Friday traffic, the recommendation engine becomes overloaded and starts responding slowly, causing timeouts.

Without circuit breakers, the main application would continue trying to fetch recommendations, tying up threads and potentially making the entire site unresponsive. With circuit breakers, after detecting failures in the recommendation service, the breaker opens and the main application serves product pages without recommendations, maintaining core functionality while the recommendation service recovers.

### Financial Trading System

In high-frequency trading systems, market data services provide real-time price feeds that trading algorithms depend on. If a market data service becomes unavailable or slow, trading algorithms need to make decisions quickly without waiting for timeouts.

Circuit breakers allow trading systems to quickly detect market data service failures and switch to alternative data sources or cached data, ensuring that trading decisions can continue to be made even when some data sources are unavailable. This prevents missed trading opportunities and maintains system responsiveness during critical market events.

### Social Media Platform

Social media platforms often integrate with external services for features like link previews, location services, or content analysis. When these external services experience issues, they shouldn't impact the core social media functionality.

Circuit breakers protect against slow or failing external services by quickly detecting problems and serving fallback responses. For example, if a link preview service is down, the circuit breaker can serve posts without previews rather than making users wait for timeouts or causing the entire feed to become unresponsive.

### Microservices Architecture

In a complex microservices environment, services have intricate dependency graphs where Service A depends on Services B and C, which in turn depend on Services D, E, and F. When Service E fails, it can potentially impact multiple services up the dependency chain.

Circuit breakers at each service boundary prevent failures from propagating upward through the dependency graph. When Service E fails, the circuit breaker protecting calls to Service E opens, allowing Service C to provide degraded functionality instead of becoming completely unavailable, which protects Services A and B from being affected by Service E's problems.

## Simple Circuit Breaker Implementation

```python
import time
import threading
from enum import Enum
from typing import Callable, Any, Optional

class CircuitState(Enum):
    CLOSED = "closed"
    OPEN = "open" 
    HALF_OPEN = "half_open"

class CircuitBreakerError(Exception):
    """Exception raised when circuit breaker is open"""
    pass

class CircuitBreaker:
    def __init__(self, 
                 failure_threshold: int = 5,
                 failure_timeout: int = 60,
                 expected_exception: type = Exception,
                 fallback_function: Optional[Callable] = None):
        # Configuration
        self.failure_threshold = failure_threshold
        self.failure_timeout = failure_timeout
        self.expected_exception = expected_exception
        self.fallback_function = fallback_function
        
        # State tracking
        self.failure_count = 0
        self.last_failure_time = None
        self.state = CircuitState.CLOSED
        self.lock = threading.Lock()
    
    def call(self, func: Callable, *args, **kwargs) -> Any:
        """Execute function with circuit breaker protection"""
        with self.lock:
            if self.state == CircuitState.OPEN:
                if self._should_attempt_reset():
                    self.state = CircuitState.HALF_OPEN
                    print(f"Circuit breaker moving to HALF_OPEN state")
                else:
                    return self._handle_open_circuit()
            
            if self.state == CircuitState.HALF_OPEN:
                return self._handle_half_open_call(func, *args, **kwargs)
            else:
                return self._handle_closed_call(func, *args, **kwargs)
    
    def _should_attempt_reset(self) -> bool:
        """Check if enough time has passed to attempt reset"""
        if self.last_failure_time is None:
            return False
        return time.time() - self.last_failure_time >= self.failure_timeout
    
    def _handle_open_circuit(self) -> Any:
        """Handle requests when circuit is open"""
        if self.fallback_function:
            print("Circuit breaker OPEN - executing fallback")
            return self.fallback_function()
        else:
            print("Circuit breaker OPEN - failing fast")
            raise CircuitBreakerError("Circuit breaker is OPEN")
    
    def _handle_half_open_call(self, func: Callable, *args, **kwargs) -> Any:
        """Handle requests when circuit is half-open"""
        try:
            print("Circuit breaker HALF_OPEN - testing service")
            result = func(*args, **kwargs)
            self._record_success()
            return result
        except self.expected_exception as e:
            self._record_failure()
            raise e
    
    def _handle_closed_call(self, func: Callable, *args, **kwargs) -> Any:
        """Handle requests when circuit is closed"""
        try:
            result = func(*args, **kwargs)
            self._record_success()
            return result
        except self.expected_exception as e:
            self._record_failure()
            raise e
    
    def _record_success(self):
        """Record successful request"""
        self.failure_count = 0
        if self.state == CircuitState.HALF_OPEN:
            self.state = CircuitState.CLOSED
            print("Circuit breaker moving to CLOSED state - service recovered")
    
    def _record_failure(self):
        """Record failed request"""
        self.failure_count += 1
        self.last_failure_time = time.time()
        
        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN
            print(f"Circuit breaker TRIPPED - moving to OPEN state after {self.failure_count} failures")
    
    def get_state(self) -> CircuitState:
        """Get current circuit breaker state"""
        return self.state

# Example usage with a simulated service
class ExternalService:
    def __init__(self):
        self.failure_mode = False
        self.call_count = 0
    
    def make_api_call(self, data):
        """Simulate an external API call"""
        self.call_count += 1
        print(f"API call #{self.call_count} with data: {data}")
        
        if self.failure_mode:
            raise Exception(f"Service unavailable (call #{self.call_count})")
        
        return f"Success response for: {data}"
    
    def enable_failure_mode(self):
        """Simulate service going down"""
        self.failure_mode = True
        print("Service entering failure mode")
    
    def disable_failure_mode(self):
        """Simulate service recovery"""
        self.failure_mode = False
        print("Service recovered")

def fallback_response():
    """Fallback function when service is unavailable"""
    return "Fallback response - service temporarily unavailable"

# Demo circuit breaker behavior
def demo_circuit_breaker():
    service = ExternalService()
    circuit_breaker = CircuitBreaker(
        failure_threshold=3,
        failure_timeout=5,
        fallback_function=fallback_response
    )
    
    # Normal operation
    print("=== Normal Operation ===")
    for i in range(3):
        try:
            result = circuit_breaker.call(service.make_api_call, f"request_{i}")
            print(f"Result: {result}")
        except Exception as e:
            print(f"Error: {e}")
        time.sleep(1)
    
    # Service fails
    print("\n=== Service Failure ===")
    service.enable_failure_mode()
    
    for i in range(5):
        try:
            result = circuit_breaker.call(service.make_api_call, f"failing_request_{i}")
            print(f"Result: {result}")
        except Exception as e:
            print(f"Error: {e}")
        print(f"Circuit state: {circuit_breaker.get_state().value}")
        time.sleep(1)
    
    # Wait for timeout and recovery
    print("\n=== Waiting for Recovery ===")
    service.disable_failure_mode()
    time.sleep(6)  # Wait longer than failure_timeout
    
    for i in range(3):
        try:
            result = circuit_breaker.call(service.make_api_call, f"recovery_request_{i}")
            print(f"Result: {result}")
        except Exception as e:
            print(f"Error: {e}")
        print(f"Circuit state: {circuit_breaker.get_state().value}")
        time.sleep(1)

if __name__ == "__main__":
    demo_circuit_breaker()
```

## Common Interview Questions

**Q: What is the Circuit Breaker pattern and why is it important?**

The Circuit Breaker pattern is a fault tolerance mechanism that monitors service calls and prevents cascading failures by temporarily stopping requests to failing services. It's important because it protects system resources, prevents cascading failures, enables faster recovery, and provides graceful degradation during outages. Like an electrical circuit breaker, it detects failures and "trips" to protect the overall system. It operates in three states: closed (normal operation), open (blocking calls to failed service), and half-open (testing if service has recovered).

**Q: How does a circuit breaker decide when to trip and when to reset?**

Circuit breakers trip based on configurable thresholds such as failure percentage (e.g., 50% of requests failing), minimum request volume (e.g., at least 10 requests), and time windows (e.g., within last 60 seconds). They reset after a timeout period when transitioning to half-open state, where test requests determine if the service has recovered. Successful test requests close the circuit, while failures return it to open state. The key is balancing sensitivity (detecting real problems quickly) with stability (avoiding false positives from temporary glitches).

**Q: What's the difference between circuit breaker, timeout, and retry patterns?**

These patterns address different aspects of fault tolerance: Timeouts prevent hanging indefinitely on slow services but don't prevent repeated attempts to failing services. Retries attempt failed operations multiple times but can worsen problems by increasing load on failing services. Circuit breakers monitor failure patterns and prevent calls to persistently failing services, providing faster failure detection and system protection. They're often used together: timeouts detect slow services, retries handle transient failures, and circuit breakers protect against persistent failures.

**Q: How do you implement fallback strategies with circuit breakers?**

Fallback strategies provide alternative responses when circuit breakers are open: return cached data (serve stale but valid information), provide default values (empty lists, default configurations), offer degraded functionality (simplified features), return static responses (maintenance messages), or redirect to alternative services. The key is designing meaningful fallbacks that maintain user experience. For example, an e-commerce site might show products without recommendations rather than failing completely, or a social media feed might load cached content when real-time updates aren't available.

## Circuit Breaker Best Practices

### Configure Appropriate Thresholds

Set failure thresholds based on your specific service characteristics and business requirements. Consider factors like normal failure rates, acceptable downtime, and service criticality. Monitor your services to understand normal failure patterns and set thresholds that detect real problems without triggering on normal variations.

### Implement Meaningful Fallbacks

Design fallback responses that provide value to users rather than generic error messages. Consider what constitutes acceptable degraded functionality for each service and implement fallbacks that maintain core user experience. Cache frequently accessed data to serve during outages, and provide clear communication about what functionality is temporarily unavailable.

### Monitor Circuit Breaker States

Implement comprehensive monitoring and alerting for circuit breaker state changes. Track metrics like trip frequency, open duration, and fallback usage to understand system health and identify recurring problems. Use this data to tune thresholds and improve service reliability.

### Test Failure Scenarios

Regularly test circuit breaker behavior through chaos engineering and failure injection. Verify that circuit breakers trip appropriately, fallbacks work as expected, and services recover correctly. Test different failure modes including slow responses, intermittent failures, and complete service outages.

### Design for Graceful Degradation

Build systems that can operate with reduced functionality rather than complete failure. Identify critical vs. non-critical features and ensure that circuit breakers protect non-critical services from impacting critical functionality. Consider the user impact of different service failures and prioritize accordingly.

The Circuit Breaker pattern is an essential tool for building resilient distributed systems. When implemented correctly, it provides automatic fault detection, system protection, and graceful degradation that significantly improves overall system reliability and user experience during service disruptions.
