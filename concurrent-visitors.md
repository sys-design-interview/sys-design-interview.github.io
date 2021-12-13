# Number of concurrent users in a website

## Question
**Interviewer**:  We are maintaining a website, and we'd like to count the \
number of concurrent visitors to the website.

---

> **_NOTE:_**  The question above is (intentionally) very vague. The \
interviewer is expecting you to ask clarifying questions to reduce the \
scope of the problem.

---

## Clarifying questions
**Candidate**: What type of website is it? Is this a live-streaming platform \
where we need to count the number of concurrent streamers? Or is it just a \
web-page which has either some static HTML or other interactive elements in it?

**Interviewer**: Not a live-streaming platform. Similar to expedia-like site.

---

> **_NOTE:_** If this is a live-streaming website, we would have websocket \
servers maintaining connections to the each client who is watching the stream. \
So, the number of concurrent viewers would be the number of open websocket \
connections at any given time.

---

**Candidate**: What is the typical QPS (queries-per-second) to the website?

**Interviewer**: ~100k per second, globally.

**Candidate**: How is the concurrent-users number being used? Is it shown to \
the users? Is it for metrics monitoring for developers? Auditing purposes?

**Interviewer**: Mostly for developers and auditing. Showing to users is fine,\
but can be lower priority for this discussion.

**Candidate**: How accurate should the numbers be? I'm asking since you \
mentioned auditing as a use-case. Are these numbers being used to show compliance \
to a particular standard, for example?

**Interviewer**: Not being used for proving compliance. So slight inaccuracies \
will not be catastrophic. However, if the numbers are way off, this would \
eventually lead to potential revenue loss.

**Candidate**: Do we need the numbers in real-time?

**Interviewer**: Yes, since it is used for metrics monitoring, we'd like to \
see the numbers as real-time as possible.

## API
**Candidate**: Let us assume that the client calls `open_session()` and \
`close_session()` whenever the session is started or closed. In case of users \
who visit a page and remain idle (or switch to a different tab), we can detect \
those in the client itself, by having a timer for inactivity. Also, we'll \
assume that on tab-close event, we'll invoke a call to `close_session` as well.\
Does that sound ok?

**Interviewer**: Yes.

## High-level design

**Candidate**: 

***Main idea***

The key observation here is that we can view the open/close \
sessions as a stream of `+1`s and `-1`s.\
The number of active sessions is the sum of these `+1`s and `-1`s. Since \
it is just an integer, each webserver can keep incrementing/decrementing \
the current count whenever it receives `open_session` or `close_session` call.

***In memory datastructure***

Also, since each webserver is just maintaing a count, they can be stateless.\
It doesn't matter if a webserver that received an `open_session()` is different \
from a webserver that received a corresponding `close_session()` call for the \
same client. At the end, we will just sum up all the counts in each webserver,\
and render them in a monitoring dashboard.

***Persistent store***

We will use a timeseries database (typically a wide-column store) like
[InfluxDb](https://www.youtube.com/watch?v=2SUBRE6wGiA) \
or [OpenTSDB](https://www.youtube.com/watch?v=WlsyqhrhRZA)
to persist the counts at a certain granularity (typically, per-second counts).


***Rendering/Visualization***

The webservers will write the counts periodically (say, every second) to the \
timeseries DB. We can build visualization servers on top of this store \
to render the trends. In case we want to scale these servers, we can add more \
machines, and have a cache in each instance to reduce hitting the persistent DB \
directly for each query.

---
> **_NOTE:_** There are 2 options for writing to the timeseries DB here: \
push vs. pull. This depends on the type of metrics-monitoring system we choose.\
Discussing pros/cons of each approach could be a good talking point for deep-dive.

---

***"Slow" path***

In order to have auditing support, we can have a batch job (running nightly, \
or few times a day) to produce reports based on the logged events. The advantage \
of such a pipeline is increased accuracy, and more readable, customized reports \
for executives (who are different from developers, and are typically interested \
in high-level business metrics, rather than performance metrics).

The webservers will write out log events for each request, including the \
open/close session events. In order to not overwhelm the downstream systems, it \
is good practice to write these log events to a queue, and have the other systems\
read from this queue instead. The consumer filters, aggregates and processes \
data from the queue. An example of this may look like:

 * filter only open/close session events
 * group and output the events in a set of files (say, output a set of files \
   for a single day, hour etc.). The file is typically in a format that is \
   optimized for consumption by a batch job (eg, columnar format).
   
Additionally, the fact that we have an offline job has other opportunities to \
filter out unwanted events like spam etc.

We then have a mapreduce that runs periodically that takes these files as input\
and produces reports as output. These reports can be static HTML pages, which\
can be copied over to a distributed file-system like GFS.

![high-level design diagram](https://sys-design-interview.github.io/concurrent-visitors.png "High level design")

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



**Interviewer**: Looks good. Can you go into details in your design?

# Deep-dives

**Candidate**: Sure, I'd like to talk about failure scenarios, trade-offs etc. \
in the "fast"-path scenario.

**Interviewer**: Ok.

**Candidate**:

***Failure scenarios***

**Webserver crashes**

Since we are storing the counts in-memory, a webserver crash could lead to loss\
of data. Even though the counts are persisted in timeseries DB, the store \
contains counts from all instances of webservers, and is not typically built to \
be used for recovering machines during crashes.

A better option might be to use write-ahead-log (WAL) with checkpointing in the\
webservers. Whenever it receives a request, a webserver first appends to a log\
on disk. Then, the request is processed (which may lead to counter increment\
or decrement). Periodically, the webserver should also checkpoint the current\
count to disk.

At startup, a webserver could have 2 modes: `failover`, or `new_server`.\
In `failover` mode, the server looks for the latest checkpoint file, and updates\
its count at startup, then reads the WAL from the checkpoint timestamp to catch up.\
In `new_server` mode, it just starts with the in-memory\
count equal to 0.

Other options (which are inferior, IMO) include:

* Having a "shadow" webserver for each instance. When one crashes, the other can\
  take over seamlessly. This has the disadvantage of consuming twice the resource\
  footprint, just for handling the failover case.
* No failover mechanism. This results in loss of data. Could be ok if we don't \
  want to introduce extra complexity in the system, and are willing to have inaccuracy.


**Client may not send close-events properly**

We rely on the client to send the `close_session()` request and to detect timeouts.\
This may not work in some scenarios like:

* client's browser tab crashes
* client's machine gets shut down unexpectedly.

In the above scenarios, we will over-count the number of concurrent visitors,\
as the `open_session()` is always recorded, but `close_session()`s are lost.

Some strategies to mitigate this include:

_Option 1:_ Extrapolation.\
Assume that there are _X%_ of lost `close_session()` calls per unit time. Then,\
we can just adjust the dashboard to adjust the numbers when rendering the data.

In the slow-path, we will have a similar problem, where we see an `open_session` \
event, without any corresponding `close_session`. In the slow-path, we can \
mitigate this by assuming a typical session-length timeout for such cases. For \
example, based on the usage patterns, we would know that a typical session lasts\
for, say, 5 minutes. So, in the batch-job, detect those session-ids which do not\
have a corresponding `close_session`. These are likely candidates for which \
the client failed to close the session. For these candidates, inject `close_session` \
events 5 minutes after the observed timestamp for their corresponding `open_session`.\
Now, this would give us a better estimate of the number of concurrent users.

_Option 2:_ Sticky sessions + keep track of session-ids

***Core idea***

Each webserver instance maintains a list of active `session_id`s, along with a timestamp\
of the last activity in that session. When a webserver receives a request for \
new connection, it generates a session_id and appends it, along with the current \
timestamp to its `sessions_list`. The list needs to be sorted by timestamp. \
(a set datastructure might be suitable here).

***Session heart-beats from client***

The client then sends a `session_heart_beat(session_id)` every time an activity \
is detected in the browser. On receiving such a request, the webserver simply \
updates the timestamp for the corresponding session_id in its in-memory datastructure.

***Sticky sessions***

Note that the `sessions_list` is in-memory. So, every time a client communicates \
related to a user's session, we should route the request to the appropriate webserver.\
In other words, we'd need to have "sticky sessions". This should be a configuration \
in the load-balancer, which instructs it to route a client's request to the same \
machine repeatedly (instead of routing it to a random webserver, for example).\
This way, the client request always lands in the webserver that contains the client's\
session in `sessions_list` and updates it accordingly.

***Periodic clean-up of inactive sessions***

The webserver should periodically loop over the `sessions_list` and remove those \
which are beyond the average-session-timeout. This is to detect and remove the lost \
`close_session()` calls.

***Updating counts***

The webserver should also periodically flush the current size of `sessions_list` \
to the timeseries database. That way, the downstream flow of visualizing the \
metrics remains the same.


***Failover***

For being resilient to crashes, we can use WAL + checkpointing approach by \
serializing the `sessions_list` to disk. That way, the new webserver which takes \
over will be able to restore the list of session-ids. However, it is not clear \
how the load-balancer would re-route the clients back to the new webserver.

***Conclusion***

I will go with Option 1 as it is much simpler, and there are ways to get more \
accurate counts via offline jobs anyway. Option 2 relies on sticky-sessions, \
which has a few disadvantages:

* resource imbalance, as there could be some machines which accumulate a lot of\
  sessions.
* no clear failover story.
* some alternatives to fix this could be to use a central Redis-backed session-store \
  which may become a single-point of failure, and adds complexity.


# Note to readers

If you see any issues with the existing approach, or would like to suggest \
improvements, please send an email to contact@sys-design-interview.com, or comment \
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
