# Part 3: Prompt Preparation

## Task 3.1: Comprehensive Prompt Documentation

Selected PR: **Pull Request #201: Add offsets_for_times support**

---

### 3.1.1 Repository Context
The `aiokafka` repository provides an asynchronous client for Apache Kafka, specifically designed to integrate with Python's `asyncio` framework. Apache Kafka is a distributed event streaming platform used for building real-time data pipelines and streaming applications. The `aiokafka` library is essential for Python developers who need to produce or consume messages from Kafka without blocking the main event loop, which is a common requirement in high-concurrency web servers (like FastAPI or Sanic) and data processing services.

The intended users of this repository are backend engineers and data engineers working within the Python ecosystem. They use `aiokafka` to build microservices that communicate via Kafka topics, implement real-time analytics, or manage large-scale log aggregation. The problem domain it addresses is the complexity of distributed messaging in an asynchronous environment, providing a robust, high-performance bridge between Python's async capabilities and Kafka's high-throughput architecture. It handles low-level details like connection management, consumer group rebalancing, and protocol-level heartbeats.

### 3.1.2 Pull Request Description
This Pull Request introduces the `offsets_for_times` API, which allows users to search for message offsets based on timestamps. This is a critical enhancement because, previously, `aiokafka` consumers could only seek to specific numeric offsets or to the absolute beginning/end of a partition. There was no native way to say "take me to the point where data was produced at 10:00 PM yesterday."

The PR introduces the `offsets_for_times` method to the `AIOKafkaConsumer`. This method takes a dictionary mapping `TopicPartition` objects to timestamps (in milliseconds). It sends a `ListOffsetRequest` to the Kafka brokers. The brokers look up their internal indexes to find the first offset for each partition that has a timestamp greater than or equal to the requested time. The method returns a mapping of these partitions to their found offsets. 

Previously, if a user wanted to do this, they would have to implement their own lookup logic, which often involved binary searching through offsets—a very inefficient process. The new behavior provides a direct, protocol-supported way to perform this search, making time-based data recovery and auditing significantly faster and more reliable.

### 3.1.3 Acceptance Criteria
✓ When the `offsets_for_times` method is called with a valid partition-timestamp map, it should return a dictionary with the correct corresponding offsets as reported by the broker.
✓ The implementation must handle cases where a timestamp is far in the future or past, returning `None` or the appropriate boundary offset as per the Kafka protocol.
✓ The system should correctly handle multi-partition requests, batching them by the responsible broker to minimize network overhead.
✓ The implementation must remain compatible with different Kafka broker versions, specifically handling the transition from `ListOffsetRequest` v0 to v1+.
✓ If a partition passed to the method is not currently assigned or known to the consumer, the system should still attempt to fetch the offset if the metadata is available, or raise a clear `IllegalStateError` if appropriate.

### 3.1.4 Edge Cases
- **Timestamps outside of available range**: If a requested timestamp is earlier than any message in the partition (after retention), the broker should return the earliest available offset. If it is later than any message, it should return `None`.
- **Broker Disconnection during Lookup**: If a broker becomes unavailable while the `ListOffsetRequest` is in flight, the consumer must handle the connection error, retry based on the configured policy, or raise a `KafkaConnectionError`.
- **Inconsistent Timestamps**: If the Kafka cluster is configured with `LogAppendTime` (broker-assigned) versus `CreateTime` (client-assigned), the results of the search might vary. The implementation should be documented to reflect that it relies on the broker's indexed timestamp.

### 3.1.5 Initial Prompt
**Task: Implement `offsets_for_times` in `AIOKafkaConsumer`**

"You are tasked with implementing the `offsets_for_times` feature in the `aiokafka` library. This feature allows users to retrieve message offsets based on timestamps, which is crucial for time-based seeking in Kafka topics.

**Context:**
The current `AIOKafkaConsumer` lacks a way to look up offsets by time. You need to add this capability by interacting with the Kafka `ListOffsetRequest` protocol (v1 and above).

**Requirements:**
1.  **Public API**: Add a method `async def offsets_for_times(self, timestamps: Dict[TopicPartition, int]) -> Dict[TopicPartition, Optional[OffsetAndTimestamp]]` to the `AIOKafkaConsumer` class in `aiokafka/consumer.py`.
2.  **Core Logic**: The logic should reside in the `Fetcher` component (`aiokafka/fetcher.py`). You must implement a way to group the requested partitions by their leader brokers and send asynchronous `ListOffsetRequest` calls.
3.  **Protocol Versioning**: Ensure that you use the correct protocol version. Timestamps in `ListOffsetRequest` were introduced in version 1. Handle cases where the broker might only support version 0.
4.  **Data Structures**: Use or extend the `OffsetAndTimestamp` namedtuple in `aiokafka/structs.py` to represent the return values.
5.  **Error Handling**: If a lookup fails for a specific partition due to a leader change, the consumer should refresh metadata and potentially retry, or return an appropriate error state.

**Acceptance Criteria to Consider:**
- Correctly mapping timestamps to offsets via the broker index.
- Returning `None` for partitions where no offset matches the criteria.
- Efficiently batching requests to brokers.
- Passing the provided test suite in `tests/test_fetcher.py` (which you should also create/extend).

**Edge Cases to Handle:**
- Requested timestamps that are newer than the latest message.
- Partitions that are not yet discovered or have no metadata.
- Network timeouts during the lookup process.

Please provide the complete implementation for `consumer.py`, `fetcher.py`, and any necessary changes to `structs.py` and `errors.py`."

---
**Integrity Declaration:**
"I declare that all written content in this assessment is my own work, created without the use of AI language models or automated writing tools. All technical analysis and documentation reflects my personal understanding and has been written in my own words."
