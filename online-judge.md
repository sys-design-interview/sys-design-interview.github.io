# Online judge (like leetcode, hackerrank, codechef)

## Question
Design an online judge like leetcode.

## Table of contents

## Clarifying questions

**Candidate**: I'm familiar with such systems. Can we focus on designing the \
code-focused scenarios?

**Interview**: Can you list the scenarios?

**Candidate**: Here's what I was thinking:

* submitting solution
* Result of submission is shown when it's ready
* viewing history of all submissions for the user
* viewing history of all submissions for a particular question for the user.

Not in scope (lower priority):
* problem-setter's flow
    * uploading new problem
    * creating contest and leaderboard
* problem recommendations

**Interviewer**: Ya, that looks ok to me.

**Candidate**: Do you have an estimate for the number of submissions coming in \
per second during peak times?

**Interviewer**: We should be able to handle a peak load of about 100 submits \
per second.

**Candidate**: Ok. Let us decide how the evaluation is done by the online judge.\
The basic idea behind evaluation is this:

* we assume that the problem-setter has provided a `test_case_file.txt` and a \
  corresponding `correct_answers.txt` file for each problem.
* the coder should assume that the input is provided via `stdin`
* each problem statement provided by the setter should specify clearly the input \
  and the format in which the input will be provided.
* given the above, the evaluation is conceptually as simple as these 2 steps (assuming \
  user chose python as their programming language):
    * `python user_submitted_file.py < test_cases_file.txt > user_solution.txt`
    * compare `correct_answers.txt` and `user_solution.txt` to see if all outputs
      match.

**Interviewer**: Hmmm, another option is to make the coder to implement a specific \
class with a method-name and arguments. But your proposal looks good to me for now.

## High level design

**Candidate**: *(just thinking out loud)* One of the main parts of the flow is \
the code-submit part, where the coder uploads their submission. This needs to be \
stored somewhere. A distributed filesystem is a natural choice here, as we can \
just store the user-submissions as files. The distributed filesystem can take \
care of replicating the data under the hood.

Another thing to note here is that it takes a lot of time to evaluate the submission. \
So, having the server synchronously compile, run and evaluate the user code is not \
ideal. We'll have to do this asynchronously.

Having those things in mind, here's what the high-level design might look like.

[Link to excalidraw](https://sys-design-interview.github.io/online-judge.excalidraw)

![Online-judge high-level-design](https://sys-design-interview.github.io/online-judge.png)

**Interviewer**: Why do you need controller and the queue to append your \
work items?

**Candidate**: We'd need that to prevent all workers from constantly hitting the \
persistent store for pending work. If the controller is not present, the alternative \
approach would be:

* each worker instance reads the persistent store and selects 1 `PENDING` item.
    * NOTE: since multiple workers would be reading the persistent store in parallel, \
      this would need to have some sort of a read lock on the database to make these \
      operations linearizable. That might lead to a bottleneck.
* after getting the `PENDING` item, the worker sets it to `PROCESSING`.
* the above 2 steps happen inside a transaction. That way the selection of work-item \
  and change in status happen atomically.

The above approach creates a bottleneck on the persistent store. With the controller \
approach, there is exactly one component polling for pending items. After selecting \
all `PENDING` items, we can use the queue + multiple-workers setup to parallelize \
the processing on those pending items.

Of course, this set up adds additional components, which require us to handle \
failure scenarios. We will get to that in a bit.

**Interviewer**: Ok.

## Components' descriptions and responsibilities

**Candidate**: Let us go over the responsibilities of each component in the design.

### Code submission

1. First, the client library POSTs the code contents as payload to the `Submit service`.
2. `Submit service` uploads the code to distributed filesystem (DFS). If this step \
   fails, an error is reported back to the coder.
3. Now, we'll have the path to DFS where the code resides. The `Submit service` \
   will now persist this metadata (user-id, question-id, path-to-code, submission-time) \
   to the persistent store.
   * when a new entry for a submission is added, its status is set as `PENDING`

### Controller
Now that the submission has been recorded in the persistent store, we need to \
detect the new submission and work on it.

`Controller` is the component that:

* polls the persistent store for `PENDING` work-items (ie, detect new submissions)
* optionally performs sanity checks (eg, blocked user, rate-limiting etc.)
* adds the work-item into the worker-queue
    * based on the user-id, we can enqueue the item into a dedicated "premium" \
      queue, or a free-tier queue.
* after enqueueing successfully, updates the work-item's status to `ENQUEUED` \
  in the persistent store.


### Queue(s)
These contain the pending work-items (code submissions) that need to be evaluated.\
The main benefits of the queue are:
* parallelism: Multiple workers dequeue items from the queue. Each worker can \
  independently act on the dequeued item.
* resource isolation for premium users: we could have a separate set of workers \
  for "premium" users, with added benefits. (eg, we could have a larger pool of \
  workers to reduce latency, more powerful machines etc.)

### Workers
The worker does the following:
* subscribes to the queue
* dequeues items from the queue
* evaluate the code by compiling, running and comparing outputs.
    * pulls in the code contents from the distributed filesystem
* updates the result of the evaluation in the persistent store.

### Properties of the "queue"
For the design to work properly, we'd need the queue to satisfy certain properties:

* it should support persistence of items (it cannot be a simple in-memory queue). \
  This is so that we are resilient to failures.
* it should support "leasing" support on each individual item. Specifically, each \
  worker should be able to get a "lease" whenever it dequeues an item. If needed, \
  the worker should be able to extend the "lease" if the item is still being \
  processed. This helps us detect inactive/dead worker after it has dequeued an \
  item. If the lease is not extended, the item can be assigned to another worker.
* once an item is "leased" out to a worker, it shouldn't be assigned to another \
  worker as long as the lease is active. This ensures that we won't have multiple \
  workers acting on the same item.
* (not a strict requirement) ideally, we'd want the queue to have de-duplication \
  support. That is, if the same work-item is being enqueued multiple times, it \
  should de-dupe them and act as if it was enqueued only once. However, this is \
  not a strict requirement, because our evaluation process is idempotent.

A simple choice for a queue with those properties could be Amazon SQS. It persists \
messages, and allows us to set "visibility timeout" on a per-item basis. This \
would be similar to the "lease" mechanism describe above.
