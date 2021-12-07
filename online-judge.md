# Online judge (like leetcode, hackerrank, codechef)

## Question
Design an online judge like leetcode.

## Table of contents
* [Clarifying questions](#clarifying-questions)
* [High level design](#high-level-design)
* [Components' descriptions and responsibilities](#components-descriptions-and-responsibilities)
    * [Code submission](#code-submission)
    * [Controller](#controller)
    * [Queue](#queue)
    * [Workers](#workers)
    * [Properties of the "queue"](#properties-of-the-queue)
 * [Data model](#data-model)
 * [Failure scenarios](#failure-scenarios)
 * [Controller crashes/fails](#controller-crashesfails)
 * ["Enqueue" operation fails](#enqueue-operation-fails)
 * [Talking point: At-least-once delivery property of the queue](#talking-point-at-least-once-delivery-property-of-the-queue)
 * [Ensuring a work-item is processed by only one worker at a time](#ensuring-a-work-item-is-processed-by-only-one-worker-at-a-time)
 * [Worker failures](#worker-failures)
 * [Talking point: Monitoring](#talking-point-monitoring)
 * [Sandbox (isolation, protection from malicious programs)](#sandbox-isolation-protection-from-malicious-programs)
 * [Note to readers](#note-to-readers)

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

<!-- Begin Mailchimp Signup Form -->
<link href="//cdn-images.mailchimp.com/embedcode/classic-10_7.css" rel="stylesheet" type="text/css">
<style type="text/css">
	#mc_embed_signup{background:#fff; clear:left; font:14px Helvetica,Arial,sans-serif; }
	/* Add your own Mailchimp form style overrides in your site stylesheet or in this style block.
	   We recommend moving this block and the preceding CSS link to the HEAD of your HTML file. */
</style>
<div id="mc_embed_signup">
<form action="https://github.us20.list-manage.com/subscribe/post?u=37da8c24b585e6a4dd6785111&amp;id=f35e3128e2" method="post" id="mc-embedded-subscribe-form" name="mc-embedded-subscribe-form" class="validate" target="_blank" novalidate>
    <div id="mc_embed_signup_scroll">
	<h2>Subscribe</h2>
<div class="indicates-required"><span class="asterisk">*</span> indicates required</div>
<div class="mc-field-group">
	<label for="mce-EMAIL">Email Address  <span class="asterisk">*</span>
</label>
	<input type="email" value="" name="EMAIL" class="required email" id="mce-EMAIL">
</div>
	<div id="mce-responses" class="clear">
		<div class="response" id="mce-error-response" style="display:none"></div>
		<div class="response" id="mce-success-response" style="display:none"></div>
	</div>    <!-- real people should not fill this in and expect good things - do not remove this or risk form bot signups-->
    <div style="position: absolute; left: -5000px;" aria-hidden="true"><input type="text" name="b_37da8c24b585e6a4dd6785111_f35e3128e2" tabindex="-1" value=""></div>
    <div class="clear"><input type="submit" value="Subscribe" name="subscribe" id="mc-embedded-subscribe" class="button"></div>
    </div>
</form>
</div>
<script type='text/javascript' src='//s3.amazonaws.com/downloads.mailchimp.com/js/mc-validate.js'></script><script type='text/javascript'>(function($) {window.fnames = new Array(); window.ftypes = new Array();fnames[0]='EMAIL';ftypes[0]='email';fnames[1]='FNAME';ftypes[1]='text';fnames[2]='LNAME';ftypes[2]='text';fnames[3]='ADDRESS';ftypes[3]='address';fnames[4]='PHONE';ftypes[4]='phone';fnames[5]='BIRTHDAY';ftypes[5]='birthday';}(jQuery));var $mcj = jQuery.noConflict(true);</script>
<!--End mc_embed_signup-->



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


### Queue
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


## Data model

We'll have the following entities:


|`User` table|
|-|
|user\_id (PK)|
|name|
|email|


|`Questions` table|
|--|
question\_id (PK)
path\_to\_content\_file
path\_to\_test\_cases\_file
path\_to\_solutions\_file


|`UserSubmissions` table|
|--|
user\_id (FK)
submission\_id (PK)
question\_id (FK)
submission\_timestamp (indexed)
path\_to\_code\_contents
status (`PENDING`, `ENQUEUED`, `WRONG_ANSWER`, `ACCEPTED`, `TLE`, ...)
last\_update\_timestamp


The `Controller` and `Worker` components would be mainly working on the `UserSubmissions` \
table, as it contains all the "work-item" information.

Given that the number of submits per second is going to be ~100, it seems ok to \
have a MySQL database for our system.

An alternative set up would be to have the `UserSubmissions` table be a NoSQL database, \
as that is the table which receives most number of writes. One possible way to set \
up the key for the NoSQL db would be:

Key: `<user_id>`-`<submission_timestamp>`-`<question_id>` \
Value (column-families): status, path\_to\_code\_contents, last\_updated\_timestamp

With the above key, we can do a quick range-scan for a user's submissions over a \
time-period. That helps with efficiently showing the submission history.

Having said that, I would lean towards having MySQL as my choice for DB, given the \
low write-QPS.

## Failure scenarios

### Controller crashes/fails
In order to be more resilient to controller failures, we should have 3 (or 5) replicas \
of the controller running. At any given time, the "master" controller is elected \
via master election. If the master fails, we can run another round of election \
to elect another new controller. That way, the new controller can resume operations \
quickly. Only the master will be able to enqueue items to the queue.

### "Enqueue" operation fails
If an "enqueue" on the queue fails, the controller can retry with exponential \
backoff. If it still fails after multiple attempts, we can update the persistent \
store status of this work item to `INTERNAL_ERROR` and have alerts on these items.

### Talking point: At-least-once delivery property of the queue
Our choice of queue is SQS, which guarantees at-least-once delivery. This means \
that there could be duplicate items inserted into the queue. But then, this is ok \
as long as there are not too many of these duplicates. The worst that could happen \
is that we would end up re-processing the same work-item multiple times. Given the \
evaluation process done in the worker is idempotent, the correctness will not be \
affected.

### Ensuring a work-item is processed by only one worker at a time
We will use the "visibility timeout" feature of the queue to make sure that any \
work-item that de-queues the item will not be "visible" to other workers.

### Worker failures
After a worker picks up an item from the queue, it runs a background thread to \
keep "extending the lease" by small amounts (say, 10 minutes). This serves 2 purposes:

* the message remains "invisible" to other workers, and hence prevents other workers \
  from picking up the same work-item.
* the small "lease extensions" act as "heartbeats" of the worker. If a worker crashes \
  for some reason, its lease on the work-item will expire, which makes it "visible" \
  to other workers. This way, the work-item will be acted on by another worker.

NOTE: the "lease" can be seen as just manipulating the "visibility timeout" of the \
message in the queue.

### Talking point: Monitoring
In order to monitor the overall end-to-end latency, and other failure cases, we \
can have an optional service that reads the persistent store and computes reports \
and exports metrics for end-to-end latency, failures etc.

## Sandbox (isolation, protection from malicious programs)

Each worker should run in a sandbox environment for the following reasons:
* to prevent itself from malicious scripts being submitted by the coders.
* catch and terminate programs that exceed time-limit.
* catch programs that exceed memory limit (disk output limits etc.)

This is a whole topic in itself, and we may not be able to go too deep into this \
topic. But some high-level options could be:

* Use VMs which have specified amount of RAM/disk/CPU
    * Pros: resource management is easy, isolation is easy to achieve.
    * Cons: VMs have a high overhead performance-wise, and may not be quick.
* Use `SIGALARM` for time-limit-exceeded.
    * Have a C++ program that spawns a new process to execute the incoming code.
      We can terminate the process once we have an alarm that triggers after a \
      predefined time-limit. The driver code could then report an error and update \
      status accordingly.
* Use `setrlimit` for resource limits. (available in C/C++).

# Note to readers

If you see any issues with the existing approach, or would like to suggest \
improvements, please send an email to sd.interviewprep@gmail.com, or comment \
below. Always ready to accept feedback and improve!

<!-- Begin Mailchimp Signup Form -->
<link href="//cdn-images.mailchimp.com/embedcode/classic-10_7.css" rel="stylesheet" type="text/css">
<style type="text/css">
	#mc_embed_signup{background:#fff; clear:left; font:14px Helvetica,Arial,sans-serif; }
	/* Add your own Mailchimp form style overrides in your site stylesheet or in this style block.
	   We recommend moving this block and the preceding CSS link to the HEAD of your HTML file. */
</style>
<div id="mc_embed_signup">
<form action="https://github.us20.list-manage.com/subscribe/post?u=37da8c24b585e6a4dd6785111&amp;id=f35e3128e2" method="post" id="mc-embedded-subscribe-form" name="mc-embedded-subscribe-form" class="validate" target="_blank" novalidate>
    <div id="mc_embed_signup_scroll">
	<h2>Subscribe</h2>
<div class="indicates-required"><span class="asterisk">*</span> indicates required</div>
<div class="mc-field-group">
	<label for="mce-EMAIL">Email Address  <span class="asterisk">*</span>
</label>
	<input type="email" value="" name="EMAIL" class="required email" id="mce-EMAIL">
</div>
	<div id="mce-responses" class="clear">
		<div class="response" id="mce-error-response" style="display:none"></div>
		<div class="response" id="mce-success-response" style="display:none"></div>
	</div>    <!-- real people should not fill this in and expect good things - do not remove this or risk form bot signups-->
    <div style="position: absolute; left: -5000px;" aria-hidden="true"><input type="text" name="b_37da8c24b585e6a4dd6785111_f35e3128e2" tabindex="-1" value=""></div>
    <div class="clear"><input type="submit" value="Subscribe" name="subscribe" id="mc-embedded-subscribe" class="button"></div>
    </div>
</form>
</div>
<script type='text/javascript' src='//s3.amazonaws.com/downloads.mailchimp.com/js/mc-validate.js'></script><script type='text/javascript'>(function($) {window.fnames = new Array(); window.ftypes = new Array();fnames[0]='EMAIL';ftypes[0]='email';fnames[1]='FNAME';ftypes[1]='text';fnames[2]='LNAME';ftypes[2]='text';fnames[3]='ADDRESS';ftypes[3]='address';fnames[4]='PHONE';ftypes[4]='phone';fnames[5]='BIRTHDAY';ftypes[5]='birthday';}(jQuery));var $mcj = jQuery.noConflict(true);</script>
<!--End mc_embed_signup-->




<script src="https://utteranc.es/client.js"
          repo="sys-design-interview/sys-design-interview.github.io"
          issue-term="title"
          theme="github-light"
          crossorigin="anonymous"
          async>
</script>

<head>
<!-- Global site tag (gtag.js) - Google Analytics -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-LDBDG3X3CZ"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-LDBDG3X3CZ');
</script>
</head>
