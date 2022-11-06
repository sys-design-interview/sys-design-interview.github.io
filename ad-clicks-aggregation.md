# Ad click events aggregation

## Question

**Interviewer**: Imagine we are serving ads via Facebook/Google. These ads get a\
lot of clicks from viewers. We want to aggregate these clicks and provide stats\
about how many clicks we get over certain periods of time.

---

> **_NOTE:_**  The question above is (intentionally) very vague. The \
interviewer is expecting you to ask clarifying questions to reduce the \
scope of the problem.

---

## Clarifying questions

**Candidate**: *(understand the user flow, and whom you are designing for)* Who\
will be querying for such counts/stats? Is it customer-facing, or is it used by\
developers (ie, internal to the company)?

**Interviewer**: For this discussion, let us assume that this will be internal\
only.

**Candidate**: *(identify scope, try to understand the most important part of\
the design)* Ok, then I will assume that there are no bad actors trying to hack\
or access sensitive data. What is the source of these click events? Can I assume\
that these events are processed and ingested into a distributed queue already?\
Or is that design also in scope?

**Interviewer**: Click processing and log ingestion are not in scope. You can\
assume that the data is available as part of a distribute queue.

**Candidate**:  What is the usage pattern of the aggregated click-events data?\
I'd assume the data should be visualized for querying over a particular\
timeframe. Is it being used by any other downstream system, like billing etc.?\

**Interviewer**: Only the aggregation and visualization are in scope for this\
discussion. We can discuss potential extensions if we have time.

**Candidate**: Ok, given that this is internal-only, and that there is no\
immediate downstream dependencies, can I assume that small inaccuracies are ok\
in our pipeline?

**Interviewer**: Can you elaborate on what you mean by small inaccuracies?\
*(oops, terms like "accuracy", "availability", "consistency" can be very vague\
and can have a different meaning based on context. Use these terms carefully.\
Always quantify)*

**Candidate**: When querying the aggregated data, the counts for num-clicks may\
be off by, say +/-0.001% (this is a hand-wavy number. We can discuss more when\
we talk about the design). I wanted to see if we can trade-off accuracy for\
throughput/performance, if needed.

**Interviewer**: Ya, if it comes to that, we can certainly consider trading off\
accuracy to some extent.

**Candidate**: *(trying to see what the main bottleneck of the system could be.\
Is it read-heavy/write-heavy/compute-intensive/storage-intensive?)* You said\
that the clicks are available in the queue already. How many events do we\
typically get?

**Interviewer**: 10 billion events per day. (100k per second)

**Candidate**: Hmmm, ok. *(in order to render the aggregated data, we need to\
process and persist the counts, which could be a lot of writes. Might also be\
storage-intensive, as we need to store info about all these events)* How many\
queries per second can I expect on the aggregated data?

**Interviewer**: That will be much lesser -- somewhere around 100/second.

**Candidate**: *(query patterns will help design APIs and choose the type of\
persistent store)* Ok, sounds like a write-heavy system. What is the query\
pattern? You mentioned they would be querying for counts over a period of time.\
Should we also allow drilling down to inspect counts for each ad, for example?

**Interviewer**: That is a good question. Yes, we would need to drill down to\
each ad as well.

**Candidate**: Good to know. Can I assume that an "ad id" would be a single\
integer, or a fixed set of integers, like <CustomerId, CampaignId, AdGroupId,\
AdId>?

**Interviewer**: Yes, for this discussion, you can assume either one of your\
proposed formats.

**Candidate**: Are we aggregating only counts, or should we provide other data\
as well? The click-events should probably have a lot of information like the\
bids, price etc.

**Interviewer**: We can focus on counts for now. However, the system should be\
extensible in the future.

**Candidate**: What is the durability of aggregated data?

**Interviewer**: We should support archival data, so they should be retained for\
a few years, at least.

## API

**Candidate**: Based on the information I have, the query API seems clear to me.

`get_counts(start_timestamp, end_timestamp) -> timeseries of counts for all ads`

`get_counts_for_ad(ad_id, start_timestamp, end_timestamp) -> timeseries of counts for *ad_id*`

The above query patterns also indicate that our persistent store should have\
timestamp and ad\_id as part of their keys.

## High-level design


