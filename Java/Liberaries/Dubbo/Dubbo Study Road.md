---
sub-type: Dubbo
type: Java
---
Based on the provided Dubbo learning roadmap, here is the refined, consolidated version in English, with keywords for each knowledge point.

## 01-Basic Principles & Core Architecture

### Core Concepts of RPC

- Definition and Goals of RPC | Key Words: RPC, Remote Procedure Call
    
- Client Stub's Role in RPC | Key Words: Client Stub, Marshalling
    
- Server Skeleton's Role in RPC | Key Words: Server Skeleton, Unmarshalling
    
- Common Challenges for RPC Frameworks | Key Words: Network Latency, Failure Handling
    
- Invocation Patterns: Synchronous vs. Asynchronous | Key Words: Sync, Async, Blocking, Non-blocking
    

### Dubbo Ecosystem & Core Roles

- Core Roles in the Ecosystem (Provider, Consumer, Registry, Monitor, Container) | Key Words: Provider, Consumer, Registry, Monitor, Container
    

### Dubbo Service Call Lifecycle

- Lifecycle of a Service Call (Registration, Subscription, Address Push, Direct Connection) | Key Words: Lifecycle, Register, Subscribe, P2P
    

### Dubbo Application Configuration Methods

- Classic Configuration using XML | Key Words: XML, Spring
    
- Modern Configuration with Annotations & Spring Boot | Key Words: Annotations, Spring Boot, Starter
    
- The Significance of Evolving from XML to Annotations | Key Words: Evolution, Convention over Configuration
    

## 02-Core Engine & Internal Mechanisms

### Service Registration & Discovery

- Client-Based Discovery Pattern | Key Words: Service Discovery, Client-Based
    
- Registry as a High-Availability Key-Value Store | Key Words: Registry, Zookeeper, Nacos
    
- Dynamic Service Awareness with Ephemeral Nodes | Key Words: Ephemeral Node, Watcher
    

### Dubbo 3 Application-Level Service Discovery

- Limitations of Dubbo 2's Interface-Level Discovery | Key Words: Interface-Level, Scalability
    
- Principles of Dubbo 3's Application-Level Discovery | Key Words: Application-Level, Performance
    
- The Role of the Metadata Center | Key Words: Metadata Center
    
- Alignment with Cloud-Native Infrastructure | Key Words: Cloud-Native, Kubernetes, Service Mesh
    

### Dubbo Protocol Stack

- The Ten Logical Layers of the Protocol Stack | Key Words: Protocol Stack, Layered Architecture
    
- Default Dubbo Protocol Characteristics (Header, Payload, Limitations) | Key Words: Binary Protocol, Header, Payload
    

### Thread Model & Dispatcher

- Design of Separating I/O and Business Threads | Key Words: Thread Model, I/O Threads, Business Threads
    
- Responsibilities of I/O Threads | Key Words: Netty, NIO, EventLoop
    
- The Function of the Business Thread Pool | Key Words: Thread Pool, Task Queue
    
- Dispatcher Strategies (all, direct, message, execution) | Key Words: Dispatcher, Thread Dispatch
    

## 03-Advanced Governance & Service Quality

### Cluster Tolerance Strategies

- Overview of Fault Tolerance Strategies (Failover, Failfast, Failsafe, Failback, Forking, Broadcast) | Key Words: Cluster, Fault Tolerance, Idempotency
    
- Selecting Strategies based on Business Scenarios | Key Words: Strategy Selection, Use Case
    

### Load Balancing Algorithms

- Overview of Load Balancing Algorithms (Random, RoundRobin, LeastActive, ConsistentHash) | Key Words: Load Balancing, Weight, Active Connections
    
- Selecting Algorithms based on Service Characteristics | Key Words: Algorithm Selection, Caching
    

### Service Degradation & Mocking

- Definition and Purpose of Service Mocking | Key Words: Mock, Service Degradation
    
- Application in Development and Testing | Key Words: Testing, Dependency Isolation
    
- Application in Fault Tolerance and Failure Isolation | Key Words: Fault Tolerance, Cascade Failure
    
- Configuration Methods for Service Mocking | Key Words: Mock Configuration
    

### Filter Chain Mechanism

- Filter as an AOP Interceptor Chain | Key Words: Filter, AOP, Interceptor
    
- Dubbo's Core Built-in Filters | Key Words: Built-in Filters
    
- Steps to Implement a Custom Filter | Key Words: Custom Filter
    
- Registering and Activating a Filter via SPI | Key Words: SPI, Extension
    
- Practical Example: A Custom Authentication Filter | Key Words: Authentication, Authorization
    

## 04-Production Readiness & Cloud-Native Integration

### Core Performance Tuning

- Business Thread Pool Tuning | Key Words: Thread Pool, corethreads, queues
    
- Serialization Protocol Selection (Hessian2, Kryo, FST, Protobuf) | Key Words: Serialization, Performance, Cross-language
    
- Key Tuning Parameters (timeout, connections, payload) | Key Words: Timeout, Connections, Payload
    

## 05-Ultimate Extensibility & Source Code Insight

### Dubbo SPI Mechanism

- SPI as the Cornerstone of the Dubbo Framework | Key Words: SPI, Service Provider Interface
    
- Enhancements of Dubbo SPI over Java's Native SPI | Key Words: Dubbo SPI, IOC, AOP
    
- Core Working Principles of SPI | Key Words: ExtensionLoader
    
- The Principle of Adaptive Extensions | Key Words: Adaptive Extension, URL
    

### Core Source Code Abstraction Models

- Core Abstraction Models (Invoker, Directory, Router, Cluster) | Key Words: Invoker, Directory, Router, Cluster
    

### Consumer Call Source Code Path

- Tracing the Consumer Call Path (Proxy, ClusterInvoker, LoadBalancer, Filter, Protocol) | Key Words: Source Code, Call Chain
    

### Framework Extension Practice

- Writing a Custom Load Balancer | Key Words: Custom Extension, LoadBalance
    
- Implementing the LoadBalance Interface | Key Words: Interface Implementation
    
- Registering the Extension via SPI | Key Words: SPI Registration
    
- Activating the Extension in Consumer Configuration | Key Words: Configuration, Activation