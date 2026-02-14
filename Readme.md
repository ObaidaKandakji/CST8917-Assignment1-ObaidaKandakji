# Assignment 1: Serverless Computing - Critical Analysis

### Obaida Kandakji, 041272028, CST8917

## Main Argument
- The paper says today’s “serverless computing” is mostly **Functions-as-a-Service (FaaS)**: developers upload functions and the cloud runs them when triggered.
- This is **“one step forward”** because FaaS provides **autoscaling** and **pay-as-you-go** pricing. You don’t provision servers, and capacity grows/shrinks with demand.
- It is **“two steps back”** because current FaaS designs do not match modern needs for **data-heavy** and **distributed** computing. They often make common system tasks slower, more complex, and more expensive.

## Key Limitations of First-Generation FaaS
- **Execution time constraints**
  - Functions are **short-lived**.
  - “Warm starts” may happen, but there is **no guarantee**, so you cannot depend on in-memory state across calls.
- **Communication and network limitations**
  - **I/O bottlenecks:** network throughput per function is limited and can drop when many functions share resources.
  - **No direct addressability:** functions are not stable network endpoints, so you can’t send messages to a specific running instance.
  - **No stickiness:** requests may land on different instances each time, so stateful interaction is awkward.
- **Communication forced through slow storage**
  - Since direct messaging is not supported, coordination often goes through storage/queues.
  - This adds **high latency** compared to normal networking and can become **costly** at large scale.
- **“Data shipping” anti-pattern**
  - FaaS often **pulls data to compute** instead of running compute near the data.
  - This increases **latency**, uses more **bandwidth**, and raises costs for iterative workloads.
- **Limited hardware access**
  - FaaS typically provides only CPU and limited RAM.
  - No easy access to **GPUs** or other specialized hardware, which limits performance for many modern workloads.
- **Distributed + stateful workloads are hard**
  - Distributed protocols need frequent, low-latency communication.
  - If every “message” becomes a storage read/write, distributed computing becomes too slow and inefficient.

## Proposed Future Directions
- **Fluid code and data placement:** allow the platform to colocate computation with data when helpful, while keeping elasticity.
- **Heterogeneous hardware support:** make accelerators available under autoscaling-friendly scheduling and pricing.
- **Long-running, addressable virtual agents:** support persistent, named services/actors that keep useful state and communicate with near-normal network performance.
- Also suggested: better programming models for asynchronous “disorderly” execution and **SLOs** so developers can request predictable performance and cost.


# Part 2: Azure Durable Functions Deep Dive

## 1) Orchestration model

Durable Functions adds a workflow layer on top of basic Azure Functions.
- **Client function**: starts a new workflow instance and gets an **instance ID** to track status or send events.
- **Orchestrator function**: acts like the “conductor.” It describes the steps in code but avoids doing real I/O itself.
- **Activity functions**: do the actual work and return results to the orchestrator.

Compared to basic FaaS, this is not just “function A triggers function B.” The runtime manages the workflow and retries, so you don’t have to wire together queues and storage manually. That directly targets the paper’s complaint that serverless apps often become lots of small functions glued together with slow intermediaries and awkward coordination.

## 2) State management

Durable Functions makes workflows “stateful” by saving progress automatically.
- It uses **event sourcing**: every significant step is written to a history log in the **task hub** storage.
- When the orchestrator reaches an `await`, it **checkpoints** and can shut down.
- Later, the orchestrator **replays** from the beginning, re-reading the saved history so local variables rebuild to the same values. This is why orchestrator code must be **deterministic**.

This answers the paper’s criticism that first-gen FaaS forces you to treat functions as stateless and externalize state yourself. Durable Functions still stores state in storage, but the runtime turns that into a built-in feature instead of an error-prone DIY pattern.

## 3) Execution timeouts

Durable orchestrations can run far longer than a normal function timeout because they don’t need to stay in memory.
- The orchestrator runs briefly, schedules work, checkpoints, and goes idle.
- When an activity finishes, the orchestrator wakes up and replays to continue.

So the workflow can last for hours, days, or longer in real time. However, important limits still exist:
- **Each individual execution** of an orchestrator, activity, or entity function is still subject to Azure Functions timeout rules for your hosting plan.
- The Durable Functions guidance treats timeouts like failures, so long activities should be broken into smaller steps.

## 4) Communication between functions

Orchestrators and activities “communicate” through the Durable runtime, not direct networking.
- When an orchestrator calls an activity, the runtime writes a message to the **task hub**.
- A worker picks up the activity message, runs the activity, and then writes the result back as another event in the task hub history.
- The orchestrator then replays and reads that stored event as the activity’s return value.

This partially addresses the paper’s criticism that serverless components often must use slow storage intermediaries. Durable Functions still relies on storage/queues under the hood, but it standardizes the pattern and can make coordination more reliable and simpler for developers.

## 5) Parallel execution

Durable Functions supports **fan-out/fan-in** for parallel work.
- The orchestrator starts many activity calls at once, usually collecting the returned tasks/promises.
- It then waits until all finish and aggregates results.
- Because activities are normal functions, the platform can scale them out across many workers while the orchestrator stays lightweight.

This helps with the paper’s concern that serverless struggles with distributed computing: Durable gives you a built-in coordinator for parallel workloads and failure handling. But it mainly fits “workflow parallelism”. It does not fully solve low-latency, fine-grained distributed protocols because coordination still goes through the durable runtime and storage.




# Part 3 — Critical Evaluation

## 1) Communication still goes through storage, not direct networking
In Durable Functions, the runtime keeps a workflow’s source of truth in a task hub. Microsoft defines a task hub as the current state of the application in storage including pending work, and says orchestration and activity progress is continually stored so processing can resume after restarts. With the default Azure Storage provider, orchestration history is stored in an Azure Storage **History table** that contains the history events for all orchestration instances within a task hub.

Developers write await activity style code and the platform handles delivery, retries, and recovery. But the paper’s critique still applies. Durable does not turn functions into stable, directly addressable network endpoints that can exchange low-latency messages. Coordination is mediated by persisted events and queued work items, which is reasonable for coarse workflow steps but is a poor fit for distributed protocols like leader election, membership, consensus, and transaction commit. In other words, it is still storage-mediated coordination.

## 2) Data-shipping and locality remain outside the model
Durable Functions mainly fixes workflow state, not data placement. The Durable overview frames it as a feature that lets you write “stateful functions” by managing state, checkpoints, and restarts for you. Orchestrators use event sourcing and replay, which is why orchestrator code must be deterministic. These mechanisms restore control-flow state, but they do not let developers express run near this dataset, guarantee reuse of warm caches across many activities, reduce data movement, or raise per-activity I/O bandwidth by placing compute near data. Activities still pull inputs from external services, and Durable adds no workflow-level way to request GPUs or other accelerators. For big-data work, you still need separate data engines.

## Verdict
Durable Functions is useful, but it does not solve the paper’s core problems. It still relies on storage and queue-mediated coordination instead of direct, low-latency messaging between stable endpoints, so it’s a poor fit for distributed protocols like consensus, membership, or transaction commit. It also doesn’t give you a way to control data placement or run compute near data, so data-intensive work still depends on separate big-data systems. Durable functions makes workflows easier and more reliable, but it’s not the architectural leap the paper is arguing for.


## Sources

Official Microsoft documentation:
- Durable Functions overview: https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview  
- Durable Functions types and features: https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-types-features-overview  
- Durable orchestrations: https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-orchestrations  
- Orchestrator code constraints : https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-code-constraints  
- Task hubs: https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-task-hubs  
- Azure Storage provider details: https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-azure-storage-provider  
- Performance and scale: https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-perf-and-scale  
- Fan-out/fan-in pattern: https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-cloud-backup  
- Azure Functions scale and hosting: https://learn.microsoft.com/en-us/azure/azure-functions/functions-scale  


## AI Disclosure Statement

Chatgpt: summary for part 1 and grammer and formatting for rest
