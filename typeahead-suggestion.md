# Typeahead for search queries.

## Question
Design typeahead suggestions for search queries. Think Google-scale.

## Table of contents

* [Clarifying questions](#clarifying-questions)
* [API and interfaces](#api-and-interfaces)
* [Datastructures](#datastructures)
* [High level design](#high-level-design)
  * [Persistent store](#persistent-store)
  * [Indexer](#indexer)
    * [Batch mode](#batch-mode)
    * [Incremental mode](#incremental-mode)
    * [Output file formats](#output-file-formats)
  * [Suggestion service](#suggestion-service)
    * [In-memory](#in-memory)
    * [On startup](#on-startup)
    * [Periodic updates](#periodic-updates)
    * [Serving requests](#serving-requests)
  * [Prefix-counts updates](#prefix-counts-updates)
* [Possible improvements](#possible-improvements)
* [Note to readers](#note-to-readers)


## Clarifying questions

**Candidate**: Should personalized suggestions be supported?

**Interviewer**: No, that is not included in our discussion.

**Candidate**: What about bad-word filters?

**Interviewer**: Although not high priority, it would be a good extension to \
the core use-case.

**Candidate**: How many suggestions do we show to the user?

**Interviewer**: Let's say we show up to 10 suggestions.

**Candidate**: Can I assume that we start showing suggestions for prefixes of \
length 3 or above? This is because there may be too many suggestions to show \
for small 1-letter prefixes, which may not be relevant to the user.

**Interviewer**: Sure, let's go with a minimum of 3-letters for us to start showing \
suggestions.

**Candidate**: Can I assume that the ordering (and the list of suggestions) is \
based on frequency of the queries?

**Interviewer**: I'll let you decide the ranking logic.

**Candidate**: Ok. Also, I'm assuming that we are only dealing with English letters \
and numbers. Is that ok, or should we broaden the scope for other characters as \
well? If so, we might need to do some data-cleaning, probably.

**Interviewer**: Let's stick to alphanumeric characters for now.

## API and interfaces

**Candidate**: For the interface with the UI, there are couple of options. We could \
have javascript on the webpage, which listens for keystrokes on the search box. \
When the prefix length is 3 or above, we could have the javascript send HTTP requests \
to our servers with the prefix information. The server can then respond to such requests \
with the list of suggestions, which can be rendered by the UI.

Another option is to use websockets, given the 2-way communication between client \
and server. This could be faster because once we establish the websocket connection, \
there is no additional overhead to sending/receiving messages from server, which \
could help with latency.

That being said, I'll keep things simple for now, and go with the HTTP option. We \
can re-visit this later, if you feel the websocket option needs to be discussed.

**Interviewer**: HTTP requests sounds good for now.

## Datastructures

**Candidate**: For in-memory data-structure, we could consider trie, or prefix hashmaps.\
In a trie, we'd store the frequency of the prefix in the node as well. That way, \
in order to retrieve suggestions for a given prefix, we need to traverse all descendants \
of a trie-node. Given that retrieving suggestions for a given prefix is the most \
common operation, we could save some of the trie-traversals by using prefix hashmaps. \
In this option, we'd just store the entire prefix, along with the list of top suggestions.\
So, for retrieval, all we need to do is just lookup the hashmap for the given prefix,\
and we'd get the list of suggestions right away.

Given that the system is latency sensitive, I'll go with the prefix hashmaps option.

**Interviewer**: Hmmm, ok.


## High level design

[Link to image](https://sys-design-interview.github.io/typeahead-suggestion.png)

[Link to excalidraw in case you want to edit](https://sys-design-interview.github.io/typeahead-suggestion.excalidraw)

![Typeahead suggestion HLD](https://sys-design-interview.github.io/typeahead-suggestion.png)

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



### Persistent store

The persistent store is the source of truth for all prefix counts. It has a very \
simple structure.

*Key* is the prefix string.

*Value* is the count for the prefix string.

In subsequent sections, we will see how this key-value data is being converted \
into prefix hash-maps, stored and served.

Given the simplicity and the fact that there are no other complex tables in the system, \
we could use a NoSQL database which supports change-streams for this store.


### Indexer

The indexer operates in 2 modes:

1. Batch mode: during which it produces *snapshots* of the full source of truth from \
   persistent store. This is done once every few hours. This is so that the `Suggestion Service` \
   can reconstruct its state on restarts.
2. Update mode: during which it produces *incremental updates* which are computed from \
   change-streams that are coming from the persistent store. This is so that `Suggestion Service` \
   can keep up with the updates to changes in the prefix counts.
   
#### Batch mode

The main purpose of batch mode is to make sure we have periodic full snapshots of \
the entire source of truth, stored in files (called *snapshot* files). The computing \
logic would be similar to a mapreduce job that counts the number of occurrences of \
prefixes, and writes them out to files.

It has the following advantages:

* decouples the serving components (eg, `Suggestion Service`, `Frontend server`) from \
  the indexing components. The servering system only needs to read the files, and \
  never update the database.
* in case we deploy the system globally, we can copy these files to every serving \
  datacenter.

#### Incremental mode

The indexer will have to subscribe to the change-streams of the persistent store. \
This will give us information about changes in prefix-counts. The indexer can then do \
the following:

* "shuffle" and partition the change events into multiple queue topics.
  * NOTE: we should partition the indexer workers in such a way that each machine \
    can handle the prefix-counts for each queue topic in-memory.
* "micro"-batch the changes from each topic in small timeframes (say, 30 seconds)
* in each "micro"-batch, perform prefix-counts in memory (remember, we partitioned \
  so that each worker has enough memory to handle a queue topic). Imagine this to \
  be similar to a "reduce" operation in mapreduce.
* Now that all "micro"-batches are processed individually, we need to aggregate them \
  into a single file, and write it out.
  
Here's a visual representation of how the incremental mode might look like:

![indexer-inc-mode](https://sys-design-interview.github.io/typeahead-indexer-inc-mode.png)

[Link to image](https://sys-design-interview.github.io/typeahead-indexer-inc-mode.png)

[Link to excalidraw in case you want to edit](https://sys-design-interview.github.io/typeahead-indexer-inc-mode.excalidraw)


#### Output file formats

Since the snapshots and incremental-updates are ordered by time, we should encode \
that information somewhere. A simple way to represent that is to write a separate \
corresponding `.metadata` file along with each snapshot and incremental-update file.

For snapshots, the `.metadata` file could contain the time at which the snapshot was \
written out.

For incremental-updates, the `.metadata` file could contain the start and end-times of the \
data present in the file. (note that the `incremental-update` files are a steady \
stream of changes that are being sent over after processing the persistent-store's \
change-stream).

Now that we have the `.metadata` files, the serving workers could quickly decide \
which snapshot files are the latest, and which update files are relevant for them.

### Suggestion service

This is a set of machines which get prefix string as part of the request, and respond \
with list of suggestions.

#### In-memory

In memory, it contains hash-maps of the form:

`HashMap</*prefix*/ String, /*list of suggestions*/ OrderedList<SuggestionInfo>>`

The list of suggestions can have the top-K suggestions (we can set K = 20).

`SuggestionInfo` can contain the frequency of the suggested query, and other relevant info. \
For now, let us assume that the suggestion-service orders the list based on frequency-count.

Additionally, it also constructs bloomfilters of the prefixes loaded in memory. This way \
it can quickly return an empty-list for queries which have no suggestions.

#### On startup

On startup, it finds and loads the latest snapshot file into memory. 

#### Periodic updates

To keep up with changes in prefix-counts, each worker in suggestion service \
also runs a background thread looking for new updates in `incremental update` files.

The updates come in the form of `prefix, prefix_count`. So, for each \
key stored in memory, we can find if the incoming prefix fits in the top-K. This \
can be done quickly because the in-memory list is ordered. If so, we can insert \
this element, and remove the least-frequent one.


#### Serving requests

Serving requests is very simple. For each query, we just look up the in-memory map \
and return the top-K suggestions. Since the lookups are in-memory, this is will be \
a very low-latency operation.

### Prefix-counts updates

We will read the log events, which contains the full query in order to continually \
update the prefix counts.

In this sub-system, we will insert logs from each front-end server into a distributed \
queue. Given the scale of the number of incoming queries, we should have this queue \
partitioned for parallelism.

The events from the queue are consumed by batchers and aggregators, which micro-batch \
the events, aggregate counts for the micro-batch's window, and update the persistent-store \
with updated counts.

NOTE: As a possible enhancement, we could add logic in aggregators to update the \
counts in such a way the the existing value in the db is given less weightage, and \
the incoming counts are given more weightage.

This way, we have clear interactions between persistent-store and other sub-systems.

* Serving subsystem: Only reads data from files written by indexer
* Indexer subsystem: Only reads data from persistent store.
* Prefix-count-update subsystem: This is the only sub-system that mutates persistent store.

# Possible improvements

* Sharding scheme of each component (indexer, batcher/aggregator, suggestion service)
* personalization
* customized ranking (or adding ability to easily change ranking algorithm).

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
