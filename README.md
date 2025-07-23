---
layout: post
title:  "High-Performance Distributed Media Processing Dataflow: Zero-Copy And Kernel Bypass"
date:   2025-07-16 15:10:11 +1000
tags: [OS, Distributed System, Media Processing, Serialization, RPC, RDMA, ProtoBuf, Capnproto, CXL, Data Flow, Service Communication & Performance]
author: "Brian Xu" 
---

# **High Performance Media Processing Distributed Data Flow: Serialization and RPC**

# **Abstract**

This article uses common serialization protocols—**Protobuf**, **JSON**, and the **high-performance Cap'n Proto**—as a medium to introduce the **design philosophies** behind serialization tools, their application in **Media Processing Distributed Data Flow (MPDDF)**, and considerations in **technology selection** and **zero-copy optimisation** in real-world applications.

Subsequently, it discusses network transmission protocols such as **gRPC** and the high-performance **RDMA**, exploring **RDMA's design principles**, its application, and **kernel bypass optimization** within **MPDDF**.

Finally, the article presents some leading optimization techniques in related fields, including **hardware-assisted Protobuf encoder** and **disaggregated memory CXL**, offering potential directions for future exploration.

# **Introduction**

## **Media Processing Distributed Data Flow (MPDDF)**

**Distributed Media Processing System**, in industry, often refers to a scalable and developer-friendly framework designed for building and orchestrating end-to-end media processing workflows. It enables rapid development, flexible integration, and efficient deployment of video and image processing tasks, including atomic operators (e.g., enhancement, decoding), standalone processing modules (such as Lambda-style functions), and complex distributed workflows. For performance, it often supports both batch and streaming execution.

Media Processing Distributed Data Flow (MPDDF) is the data‐movement substrate at the heart of any large‑scale media processing system. It defines how media payloads—video frames, audio snippets, metadata—traverse a graph of operators (decoders, encoders, filters, analyzers) in a reliable, efficient, and schema‑aware fashion.

![](https://cdn-images-1.medium.com/max/1600/0*G4D5JOnKMOCTnw08)

The diagram depicts a classic streaming‐operator pipeline in which video frames flow continuously between streaming operators for concurrent processing.

The media‐processing engine sits atop this pipeline, handling task orchestration, container scheduling, intermediate‐artifact storage, and inter‐container messaging. Within its MPDDF layer, communication performance is paramount: every end‑to‑end handoff incurs costs from serialization/deserialization, buffer allocation and NUMA placement, one or more memory copies, system calls and kernel‑mode switches, socket or RDMA I/O, and ultimately NIC DMA and network traversal. Optimizing MPDDF means minimizing these stages so that frames advance through the pipeline with the fewest software and hardware hops possible.

# **Protobuf**

Proto is a commonly used data serialization/deserialization protocol by MPDDF in industry practice.

> _Protocol buffers are Google’s language-neutral, platform-neutral, extensible mechanism for serializing structured data – think_ _XML__, but smaller, faster, and simpler. You define how you want your data to be structured once, then you can use special generated source code to easily write and read your structured data to and from a variety of data streams and using a variety of languages._

![](https://cdn-images-1.medium.com/max/1600/0*QmZwZZ0BIC1ty_rX)

> _Why the name "Protocol Buffers"?_ _The name originates from the early days of the format, before we had the protocol buffer compiler to generate classes for us. At the time, there was a class called ProtocolBuffer which actually acted as a buffer for an individual method. Users would add tag/value pairs to this buffer individually by calling methods like AddValue(tag, value). The raw bytes were stored in a buffer which could then be written out once the message had been constructed._ _Since that time, the "buffers" part of the name has lost its meaning, but it is still the name we use. Today, people usually use the term "protocol message" to refer to a message in an abstract sense, "protocol buffer" to refer to a serialized copy of a message, and "protocol message object" to refer to an in-memory object representing the parsed message._

## **Protobuf Wire Format for Encoding**

The efficiency of Protobuf lies in its encoding format, which uses a T-(L)-V (Tag-Length-Value) structure. Each field has a unique tag that serves as its identifier. The _length_ represents the size of the _value_ data. However, _length_ is not always required — it's omitted for fixed-length values. The _value_ is the actual content of the data.

![](https://cdn-images-1.medium.com/max/1600/0*h7yF2wXiZ7zzcHke)

The _tag_ consists of two parts: `field_number` and `wire_type`. The `field_number` is the number assigned to each field in the message definition, while `wire_type` indicates the type of the field—whether it's fixed-length or variable-length. There are currently six `wire_type` values (0 to 5), which means only 3 bits are needed to represent them. The structure of a tag is illustrated below.

![](https://cdn-images-1.medium.com/max/1600/1*EM7BT6Yjf1Uu-9777VF2tg.png)

A simple message

```ProtoBuf
message SubMessage {  
  int32 c = 1;  
}  
  
message Test1 {  
  int32 a = 1;  
  string b = 2;  
  SubMessage d = 3;  
}
```

Construct in code

```JSON
Test1 {  
  a: 150,  
  b: "hi",  
  d: { c: 1000 }  
}
```

Serialized Output (in hex):

```SQL
08 96 01 12 02 68 69 1a 03 e8 07
```

**Explain:**

Encoding the Field `a = 150`

Field number = 1

Wire type = 0 (varint for int32)

**Tag** = `Field number << 3 | Wire type`

= `(1 << 3) | 0` = `0x08` **Value**

In Protobuf, integers are encoded using **Varint**, a variable-length encoding format. Varints use one or more bytes. Each byte:

-   Uses the **lower 7 bits** for data.

-   Uses the **highest bit (MSB)** to indicate if more bytes follow:

-   `1` → more bytes follow.

-   `0` → this is the last byte.




The bytes are:

`96 (hex) = 0b`**`1`**`0010110 (binary)`

`01 (hex) = 0b`**`0`**`0000001 (binary)`

Remove the MSB from each byte:

-   From `0b`**`1`**`0010110`, remove the MSB → `0b00010110` = 0x16 = **22**

-   From `0b`**`0`**`0000001`, remove the MSB → `0b0000001` = 0x01 = **1**


Reconstruct the integer (little-endian):

In Varint, the **first byte holds the least significant bits**.

So we combine the bits as:

value `= (1 << 7) | 22 = 128 + 22 = 150`

So **based on the bytes**, it's possible to **reverse-engineer the proto structure,** including the type and value of each field. Media Processing system console can use this to implement the formatted display of execution's input/output.

![](https://cdn-images-1.medium.com/max/1600/1*CnzgRXOsnQJxmuOf7rSIpQ.png)

## **Static Deserialize (with generated code)**

Static parsing means using **pre-generated code** to decode protobuf data. In Go, this is done by generating `.pb.go` files from your `.proto` files.

**When Deserialize:**

-   Uses the hardcoded field numbers, types, and structure in the generated code.

-   Parses the binary data accordingly and fills the Go struct.


## **Dynamic Deserialize (Proto Descriptor)**

A **proto descriptor** is a special protobuf message that describes the structure of a `.proto` file itself. It contains all the information about:

-   Messages, fields, enums, services

-   Field types and numbers

-   Packages and syntax (proto2/proto3)


It allows tools or programs to understand protobuf data **without needing the original** **`.proto`** **source file**.

**To generate proto descriptor:**

```Shell
protoc --descriptor_set_out=desc.pb --include_imports your_file.proto
```

Programs can parse this descriptor file at runtime, use it to generate dynamic proto message instance, and then use the instance to **deserialize** protobuf blob through **dynamic reflection**, without generated code.

Media Processing system console can use proto descriptor to convert proto blob to human-readable format for debugging purpose.

```JSON
{  
  "name":"John",   
  "age":30,   
  "car":null  
}
```

## **Benchmarking Protobuf and JSON: Performance and Size**

![](https://cdn-images-1.medium.com/max/1600/1*x00_1wEybq9W57jzq2Uvkg.png)

Performance benchmarks reveal clear differences between the two formats:

-   **Size:** Protocol Buffers produce payloads that are approximately three times smaller than equivalent JSON data. (Can be further compressed using tools like `gzip`, **reduce size by around 50%**)

-   **Serialization Speed:** Proto serializes data over ten times faster than JSON.

-   **Deserialization:** For static deserialization, Proto is also over **6 times faster** than JSON. For dynamic deserialization, both formats perform similarly.


## **Cost for Serialization/Deserialization**

Although proto buffers' serialization/deserialization is already very fast, it still requires a significant amount of CPU computation and memory overhead.

In Go, the serialization process using `bytes, err := proto.Marshal(message)` typically involves the following steps:

1.  Message Size Estimation & Buffer Allocation

1.  Traverse each field of the message to compute the encoded size.

2.  Once the total size is known, allocate a `[]byte` buffer (e.g., via `malloc`) with size at least equal to the computed size.

3.  In high-concurrency scenarios, especially when dealing with large messages, allocating buffers may trigger virtual memory expansion for the Golang processor. This can lead to system calls (e.g., `mmap`) to request virtual memory from the OS, or lead to GC, causing kernel mode switches.

4.  Allocating large size buffers also implies a higher possibility of triggering GC.

2.  Encoding Process

1.  Each field is encoded sequentially. For example, `varints` are converted into their encoded byte representations, and string/bytes fields involve memory copying. The encoder keeps moving the writing offset and writes the bytes into the buffer.

2.  For large messages, writing into a new page on the page table is likely to trigger a page fault. This causes a mode switch, and the OS will allocate physical memory.


For deserialization, the process differs slightly, but the overall overhead is largely similar.

## **Benchmarking Analysis**

On a  _**Debian 10 linux 5.4 machine with a c1.2xlarge.v6 | 8 cores | 16G**__,_ with a **1080P** video frame message, size of **3.110449 MB** after encoding (**including a 3.1104MB video frame****`[]byte`** **field** and some other meta field), w_ith a concurrency of_ _**8 qps (**__1 qps per go-routine and 8 go-routines in total, after each run, sleep 1 second__**),**_ here is the round-trip encoding time benchmark.

![](https://cdn-images-1.medium.com/max/1600/0*VKX9XROsykXoPJ9b)

![](https://cdn-images-1.medium.com/max/1600/0*03-fKfPUrRXAODnW)

![](https://cdn-images-1.medium.com/max/1600/0*asuPmfzBXgywLrC4)

Under this setting, over 100 runs, the encoding round-trip time of a 1080P video frame message is at around **7.5235ms**. This level of latency is considerable for a latency-sensitive dataflow engine system.

Performance profiling indicates that GC is not the primary contributor to the delay. Instead, the main root cause appears to be frequent **page faults** and **kernal functions (mmp)**, which introduce substantial overhead.



_**Note**_

_For my encoding round trip test, it includes the following procedures:_

1.  _`Malloc`_ _a continuous byte array in user mode memory_

2.  _Marshal the message into the byte array (process's user mode memory heap runs out, acquire memory from kernel, page fault)._

3.  _Create a pointer to a new message, with all fields empty._

4.  _Unmarshal the bytes to the new message, includes many_ _`malloc`_ _and memory copies, which is also likely to page fault._


_The message contains over 3 MB frame data._

_I think this can reflect the online scenario more realistically, and can reflect more performance differences between Capnp and ProtoBuf in real world scenarios._



**Go benchmark results from running N iterations serially in a single goroutine (with no sleep)**

![](https://cdn-images-1.medium.com/max/1600/0*Un-rmzZdeiSVCHD-)

```JSON
BenchmarkProtoAVFrame-12                4464            237764 ns/op
```

Proto Buffers encoding round trip time by go benchmark (using same message) is **237.764µs**

**Note**

To simulate 8 QPS, `time.Sleep()`was used between runs. Experiments show that this is the main reason for the performance difference between the two metric results. The exact cause remains to be further investigated, but the initial hypothesis is about context switches and CPU cache invalidation.

# **Cap'n'proto**

<img src="https://cdn-images-1.medium.com/max/1600/0*HydYq1xBPZBaehF5" width="300">

> Cap’n Proto is an insanely fast data interchange format and capability-based RPC system. Think JSON, except binary. Or think [Protocol Buffers](https://github.com/protocolbuffers/protobuf), except faster. In fact, in benchmarks, Cap’n Proto is INFINITY TIMES faster than Protocol Buffers.
>
> This benchmark is, of course, unfair. It is only measuring the time to encode and decode a message in memory. Cap’n Proto gets a perfect score because _there is no encoding/decoding step_. The Cap’n Proto encoding is appropriate both as a data interchange format and an in-memory representation, so once your structure is built, you can simply write the bytes straight out to disk!

MPDDF, as a high-throughput, latency-sensitive streaming dataflow transmission engine, responsible for handling data exchange and storage between media processing streaming operators. It can use Cap'n Proto as the serialization protocol.

Cap’n Proto uses a platform-independent, byte-for-byte encoding with fixed widths, offsets, and alignment. Pointers are offset-based (not absolute). This design ensures both portability and performance. Backwards compatibility is preserved by appending new fields to the end of structs or reusing padding space, leaving existing field positions unchanged.

## **Sample Usage (Golang)**

1.  After correctly installing Cap’n Proto, can declare IDLs in a way similar to Protocol Buffers.


```Go
using Go = import "/go.capnp";  
@0xabcdefabcdefabcdef;  
$Go.package("capnp_gen");  
$Go.import("test");  
  
struct Inner {  
  innerA @0 :UInt16;  
  innerB @1 :UInt32;  
}  
struct Outer {  
  outerA @0 :UInt16;  
  inner @1 :Inner;  
  data @2 :Text;  
}
```

2.  Generate code with IDL

3.  Create objects


```Go
arena := capnp.SingleSegment(nil)  
msg, seg, err := capnp.NewMessage(arena)  
if err != nil {  
    panic(err)  
}  
outer, err := capnp_gen.NewRootOuter(seg)  
if err != nil {  
    panic(err)  
}  
inner, err := outer.NewInner()  
if err != nil {  
    panic(err)  
}  
inner.SetInnerA(4)  
inner.SetInnerB(5)  
_ = outer.SetData("Bob")  
outer.SetOuterA(6)
```

4.  Marshal & Unmarshal


```Go
b, err := msg.Marshal()  
if err != nil {  
    panic(err)  
}  
newMsg, err := capnp.Unmarshal(b)  
if err != nil {  
    panic(err)  
}  
newOuter, err := capnp_gen.ReadRootOuter(newMsg)  
if err != nil {  
    panic(err)  
}
```

## **Core Components & Responsibilities**

![](https://cdn-images-1.medium.com/max/1600/1*8LuO8KknOJuzFUQzxX-i3w.png)

**Arena** **(Allocator Context)**

-   Manages raw memory buffers of one root message.

-   Owns a list of Segments and a backing buffer pool. And handles on-demand allocation of new Segments, (_using an exponential growth strategy to minimize copies)._


**Message**

-   Represents a single Cap’n Proto message’s API surface (getters/setters).

-   Holds a reference to its Arena, does **not** itself contain data—reads and writes are directed into the Arena’s Segments.


**Segment (Contiguous Buffer)**

-   By default, an 8-byte-aligned slice of memory (`[]byte`), treated as a sequence of 64-bit words.

-   Stores the fixed-size struct section (fields + pointer words) and the variable-length data section.

-   Each pointer word encodes a (segmentIndex, wordOffset) pair, so readers know exactly where data lives.


**Relationships**

-   **1 Arena → N Segments**: An Arena can grow to many Segments as needed.

-   **1 Message → 1 Arena**: Each Message binds to a single Arena for all its allocations.

-   **1 Segment → 1 Arena**: Segments belong exclusively to their Arena and aren’t shared across Messages.


## **Message Wire Format & Memory representation**

_Take a simple message struct with only a single segment as an example_

```Go
struct Inner {  
  innerA @0 :UInt16;  
  innerB @1 :UInt32;  
}  
struct Outer {  
  outerA @0 :UInt16;  
  inner @1 :Inner;  
  data @2 :Text;  
}  
  
Outer {  
  outerA: 6,  
  inner: {  
    innerA: 4,  
    innerB: 5  
  },  
  data: 'Bob'  
}
```

**memory representation**

```Shell
[ 0  0  0  0  1  0  2  0     6  0  0  0  0  0  0  0  
  4  0  0  0  1  0  0  0     5  0  0  0 34  0  0  0  
  4  0  0  0  5  0  0  0     66 111  98  0  0  0  0  0 ]
```

**Explain**

![](https://cdn-images-1.medium.com/max/1600/0*TlwypHgjjdcsGaPf)

![](https://cdn-images-1.medium.com/max/1600/1*iZj5_AhrSStJTz3bi6CweQ.png)


## **Inter-Segment Pointer**

When a pointer needs to point to a different segment, offsets no longer work. Instead, encode the pointer as a **far pointer**:

![](https://cdn-images-1.medium.com/max/1600/0*NNfqnDcC0EvVgGJu)

The pointer would be something that looks like this:

**`0A`** `00 00 00 |` **`01`** `00 00 00` **`0A`** `= 01 | 0 | 10` Indicates that it is a far pointer, where the landing pad offset is at 1 in segments[1]

## **Serialisation & Deserialisation**

For the native implementation of Capnp's serialisation, there's no need to encode/decode the message anymore. It allocates a new large memory bytes buffer, and copies all the raw bytes from different segments into the large buffer. Even though there is no encoding/decoding process, it still requires memory copy (likely to introduce page faults), which means it can still slow down the encoding round-trip time.

However, for the nature of Capnp's in-memory encoding representation, raw bytes can be directly deserialised between platforms. Hence, raw bytes of segments can be sent directly between services via network, without extra buffer allocation or memory copying, Capnp's Golang SDK provides encode/decode methods for achieving this (Scatter-Gather pattern).

```Go
// Server Side Code  
arena := capnp.MultiSegment(nil)  
msg, seg, _ := capnp.NewMessage(arena)  
outer, _ := capnp_gen.NewRootOuter(seg)  
outer.SetOuterA(6)  
ln, _ := net.Listen("tcp", ":12345")  
conn, _ := ln.Accept()  
encoder := capnp.NewEncoder(conn)  
encoder.Encode(msg)  
  
// Encode method is similar to:  
send([][]byte{  
    arena.Segment(0).Data(),  
    arena.Segment(1).Data(),  
    ...  
})
```

## **Benchmarking Analysis**

On exactly the same machine, same video frame message, same concurrency, but encode/decode with Capnp, here is the round-trip encoding time benchmark.

![](https://cdn-images-1.medium.com/max/1600/0*p3M9qOOF9i5iK3MH)

![](https://cdn-images-1.medium.com/max/1600/0*cL8Pus95Wm47R4YW)

![](https://cdn-images-1.medium.com/max/1600/0*rluKdfhYLDtapxIN)

Under the same setting, over 100 runs, the encoding round-trip time of a 1080P video frame message is at around **16.229µs,** with almost no page faults, in comparison with **7.5235ms** with ProtoBuf, its **463x** times faster, while the message size is only **0.00002x** times larger.



**Go benchmark results from running N iterations serially in a single goroutine (with no sleep)**

![](https://cdn-images-1.medium.com/max/1600/0*-fjpX81Z3CPXOzK6)

```JSON
BenchmarkProtoAVFrame-12                4464            237764 ns/op
```

Capnp encoding round trip time by go benchmark (using same message) is **135.5ns,** in comparison with **237.764µs** with Protobuf, its **1754x** times faster.

**Note**

To simulate 8 QPS, `time.Sleep()` was used between runs. Experiments show that this is the main reason for the performance difference between the two metric results. The exact cause remains to be further investigated, but the initial hypothesis is about context switches and CPU cache invalidation.

## **Advantages & Drawbacks**

**Advantages**

1.  **Highly performant serialization**: No need to allocate extra buffer on the heap and encode/memory copy.

2.  **Inter-process communication via shared memory**: Multiple processes running on the same machine can share a Cap’n Proto message via shared memory.

3.  **Incremental reads**: Can begin parsing before full message arrives, since outer objects appear entirely before inner objects.

4.  **Random access**: Can read just one field of a message without parsing the whole thing.

5.  **mmap:** Read a large capnp file by `mmap` it. The OS won’t even read in the parts that you don’t access.


**Drawbacks**

1.  **Larger message size**: The wire format is not compact, which can increase bandwidth usage during network transmission.

2.  **Costly updates**: Resetting **variable-sized message** fields (e.g., string and bytes), requires allocating new memory, making it less suitable for frequently mutating messages (unless directly modify raw bytes).

<pre><code>
[0 0 0 0 1 0 3 0 6 0 0 0 0 0 0 0 13 0 0 0 34 0 0 0 4 0 0 0 1 0 0 0 
9 0 0 0 58 0 0 0 4 0 0 0 5 0 0 0 <span style="color:red">66 111 98</span> 0 0 0 0 0 <span style="color:purple">66 111 98</span> <span style="color:purple">66 111 98</span> 0 0] 

[0 0 0 0 1 0 3 0 6 0 0 0 0 0 0 0 21 0 0 0 130 0 0 0 4 0 0 0 1 0 0 0 
9 0 0 0 58 0 0 0 4 0 0 0 5 0 0 0 <span style="color:gray">66 111 98</span> 0 0 0 0 0 <span style="color:purple">66 111 98</span> <span style="color:purple">66 111 98</span> 0 0 
<span style="color:red">66 111 98</span> <span style="color:red">66 111 98</span> <span style="color:red">66 111 98</span> <span style="color:red">66 111 98</span> <span style="color:red">66 111 97</span> 0]
</code></pre>


3.  **No shared substructures**: Multiple root messages cannot reference the same inner object due to its wire format.

4.  **Initial memory overhead**: Cap’n Proto allocates an 8KB segment by default, which can be wasteful unless the approximate message size is known and tuned.

5.  **Safety & debugging**: Cap’n Proto does support out-of-bounds check for safety. But pollution to pointer fields can lead to undefined behavior (such as infinite loops), which is very difficult to troubleshoot.


MPDDF, as a streaming dataflow system for media processing scenarios with high data throughput, high currency, and high latency sensitivity. It handles streaming workflows where video frames (type of []byte) are the primary source of data volume.

For video frame message types, Protobuf only applies some compression to the metadata. Based on experiments, the difference in message size between Cap’n Proto and Protobuf after serialization is negligible.

Since video frame content is typically fixed upon generation, and the data flows occur within the same data center with very low network latency, it is well-suited for a Cap'n Proto-based serialization and RPC solution.

## **Cost of RPC**

Even though Cap’n Proto is designed for high performance, its RPC still incurs fundamental system-level costs. Capnp's native RPC protocal or GRPC use traditional network data path, it needs to go through the buffers using the sockets API in the user space. In the kernel, the data path includes the TCP, IPv4/6 stack all the way down to the device driver and eventually the network fabric. All these steps require CPU cycles for processing. With high bandwidth networks today (25,40,50, and 100GbE) this can pose a challenge because of the amount of CPU time required to process data and put that data on the wire.

# **Cap'n'proto X RDMA**

> Overview Remote Direct Memory Access (RDMA) is an extension of the Direct Memory Access (DMA) technology, which is the ability to access host memory directly without CPU intervention. RDMA allows for accessing memory data from one host to another. A key characteristic of RDMA is that it greatly improves throughput and performance while lowering latency because less CPU cycles are needed to process the network packets.

![](https://cdn-images-1.medium.com/max/1600/0*Bnu8-GB5P80JdpmQ)

![](https://cdn-images-1.medium.com/max/1600/0*AKxT4Rxe4jDn0Rcn)

The core idea of RDMA is to offload key data transfer workloads from the CPU to the NIC hardware, such as memory access, transport protocol handling, and retransmission control. This reduces system calls or kernel mode transitions for CPU and increases NIC utilization. In MPDDF, RDMA can be used for transmitting Capnp message data.

## **RDMA schema - RoCEv2**

Each RDMA schema requires NIC cards with different capabilities. Here we use RoCEv2 as an example. Compared to a “plain” Ethernet NIC, a RoCEv2‑capable NIC (often called an RNIC) must support:

1.  **Queue Pair (QP) Engines**


Each RDMA session requires the creation of a **Queue Pair (QP)**, which includes and **Send Queue (SQ)** and **Receive Queue (RQ).** Applications post **Work Requests (WRs)** to these queues using `ibv_post_send()` and `ibv_post_recv()`. The **RoCEv2 NIC maintains and executes these queues in hardware** asynchronously.

2.  **Memory Registration & Address Protection**


RDMA requires the user to **register memory** onto **NIC** using `ibv_reg_mr()`, which maps virtual memory to physical addresses. The NIC manages page tables translations (the foundation for **zero-copy memory access** in RDMA).

3.  **RDMA Protocol Stack in Hardware**


The NIC has a fully offloaded RDMA verbs engine. Also handles reliable Connected (RC) mode with **hardware-based reliability.** All this is performed in hardware with no CPU involvement.

4.  **UDP/IP Header Push-down**


UDP/IP headers are generated by the NIC automatically. The NIC supports header insertion, IP/UDP checksum offload, so applications do not need to use sockets or construct UDP packets manually.

## **Data Sending Pipeline**

![](https://cdn-images-1.medium.com/max/1600/0*aTT9WMCYLji_L48Y)

## **Tunning - Completion Queue Polling**

**Polling the Completion Queue (CQ) can be done in two modes:**

-   **Polling mode:** continuously call `ibv_poll_cq()` in a loop to check for completions.

-   **Event-driven mode:** register for CQ events with `ibv_req_notify_cq()` and wait for an interrupt/event notification.


**Polling mode wastes CPU cycles** because it continuously checks for completions without blocking, while **event-driven mode requires system calls** to wait for events, introducing latency and some overhead.

A **hybrid approach** combines both: it polls the CQ actively for a certain number of times, and if no completions are found, it switches to event-driven mode to save CPU resources.

**Batch polling:** submitting multiple write requests and then poll the CQ to retrieve multiple completion entries in a single call, reducing overhead.

## **Tunning - Staging Store**

By default, Cap’n Proto allocates memory for each segment using `calloc`, which results in the OS assigning a new physical memory page to the user process.

Before sending data via an RDMA write queue, the user process must register the memory segment with the NIC via a system call, so the NIC can map the segment in its page table.

Cap’n Proto allows customizing the memory allocation strategy by overriding the `allocateSegment()` method. If there are multiple segments needing to be sent, a large shared memory buffer can be pre-allocated in advance , registered once with the NIC, and the capnp's `allocateSegment()` method is overridden to allocate segment memory via `mmap` from this shared memory. This eliminates all system calls and kernel mode switches during data transmission.

The diagram below shows one possible implementation. The Staging Store is responsible for creating the shared memory region and its **bitmap** header. The **bitmap** header divides the shared memory into a number of fixed‑size logical blocks, using one bit per block to indicate whether it is free or in use. User processes then allocate memory directly from this shared memory.

![](https://cdn-images-1.medium.com/max/1600/0*tF5qJhAoFRwYsTB-)

## **Real World MPDDF Implementation**

In the real-world implementation inside MPDDF, it can be much more sophisticated than this example. Because it may need to support failure replay and intermediate-result persistence, the Staging Store can be actually implemented as an independent distributed service. It maintains its own reference‑counting mechanism to track, release, and reuse memory blocks. In this way, the Staging Store can also decouple interaction between different streaming operators, so that the failure of one operator does not impact the others.

# **Hardware Assisted Accelerator for ProtoBuf by Google**

![](https://cdn-images-1.medium.com/max/1600/0*WXUiPBebri5xnlX9)

The Protocol Buffers hardware accelerator is a specialized module integrated into a `RISC-V` System-on-Chip (SoC), designed to offload serialization and deserialization tasks from the CPU. This integration leads to significant performance improvements, with evaluations showing an average speedup of 6.2× to 11.2× compared to a baseline `RISC-V SoC` with BOOM out-of-order cores, and a 3.8× improvement over a Xeon-based server, despite the `RISC-V SoC's` less robust supporting components .

It interfaces with the CPU via the RoCC (Rocket Custom Coprocessor) interface, allowing the CPU to dispatch custom `RISC-V`instructions directly to the accelerator with minimal latency . It has integrated with a modified version of the open-source protobuf library and maintains compatibility with standard protobuf formats

By offloading the computationally intensive tasks of serialization and deserialization, the hardware accelerator reduces CPU workload, lowers latency, and enhances overall system performance, making it particularly beneficial in data-intensive environments such as large-scale data centers  _**(Karandikar et al., 2021).**_

# **Disaggregated Memory - CXL by Intel**

![](https://cdn-images-1.medium.com/max/1600/0*t-3sxakJAcvp99CM)

Compute Express Link (CXL) as a high-performance, cache-coherent interconnect standard designed to enable memory disaggregation in data centers. By allowing CPUs to access remote memory with load/store semantics while maintaining cache coherence, CXL facilitates flexible and efficient pooling of memory resources separate from compute nodes. This architecture supports scalable, low-latency memory sharing, addressing key challenges in modern disaggregated systems. _**（Das Sharma et al., 2024）**_

In typical practical applications of MPDDF, streaming operators communicate via a third service (staging store) through RDMA. This setup is a potential scenario where disaggregated memory solutions can be utilized to enhance performance.

## **CXL integration with PolarDB by Alibaba**

At the 2025 SIGMOD/PODS Conference, Alibaba Cloud presented a paper titled "Unlocking the Potential of CXL for Disaggregated Memory in Cloud-Native Databases," introducing PolarCXLMem—a disaggregated memory system based on Compute Express Link (CXL) technology. This system aims to address the limitations of traditional RDMA-based disaggregated memory solutions, which often suffer from:

-   **Read/Write Amplification**: Inefficient data access leading to performance bottlenecks.

-   **Limited Bandwidth**: Network bandwidth constraints hindering data transfer speeds.

-   **Inefficient Recovery**: Slow data recovery processes after system crashes.

-   **Data Sharing Challenges**: Complex mechanisms for data sharing between nodes, affecting system scalability.


Performance evaluations conducted on PolarDB, Alibaba Cloud's widely deployed cloud-native database, demonstrated that PolarCXLMem can improve throughput by up to **2.1× in pooling scenarios and 1.55× in sharing scenarios compared to RDMA-based systems** _**(Yang et al., 2025).**_

These findings suggest that integrating CXL-based disaggregated memory solutions like PolarCXLMem can potentially enhance the performance and scalability of systems relying on inter-operator communication through staging stores, such as MPDDF.

## **Current Bottlenecks of MPDDF**

**Current two-stage RDMA path:** streaming operator A → (RDMA) → staging store → (RDMA) → streaming operator B, requires:

1.  Memory allocation on Staging;

2.  Two full network round trips and multiple memory copies/DMA operations;

![](https://cdn-images-1.medium.com/max/1600/1*XHUoJgUgJm1tWuyDn5qicA.png)

![](https://cdn-images-1.medium.com/max/1600/0*-_ahpW5R4nyiNA8_)

Key advantages of CXL also include, pooled memory resources, elastic memory scaling, physical resource isolation with logical access control, and improved tenant isolation, which is natively friendly to cloud service with multi-tenancy scenarios.

## **Hardware Requirements for CXL-based Disaggregated Memory**

1.  CXL-compatible CPU or Host Bridge

2.  CXL-capable motherboard or PCIe slot

3.  CXL Memory Devices (Memory Blades)

4.  CXL Switch (for multi-host sharing)


# **Reference**

Debendra Das Sharma, Robert Blankenship, and Daniel Berger. 2024. An Introduction to the Compute Express Link (CXL) Interconnect. ACM Comput. Surv. 56, 11, Article 290 (November 2024), 37 pages. https://doi.org/10.1145/3669900



Sagar Karandikar, Chris Leary, Chris Kennelly, Jerry Zhao, Dinesh Parimi, Borivoje Nikolic, Krste Asanovic, and Parthasarathy Ranganathan. 2021. A Hardware Accelerator for Protocol Buffers. In MICRO-54: 54th Annual IEEE/ACM International Symposium on Microarchitecture (MICRO '21). Association for Computing Machinery, New York, NY, USA, 462–478. https://doi.org/10.1145/3466752.3480051



Xinjun Yang, Yingqiang Zhang, Hao Chen, Feifei Li, Gerry Fan, Yang Kong, Bo Wang, Jing Fang, Yuhui Wang, Tao Huang, Wenpu Hu, Jim Kao, and Jianping Jiang. 2025. Unlocking the Potential of CXL for Disaggregated Memory in Cloud-Native Databases. In Companion of the 2025 International Conference on Management of Data (SIGMOD/PODS '25). Association for Computing Machinery, New York, NY, USA, 689–702. https://doi.org/10.1145/3722212.3724460