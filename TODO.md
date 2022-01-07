# TODO list

|Topic                                                                                                     |Status     |
|-                                                                                                         |-          |
|[Count number of concurrent users in a website](https://sys-design-interview.com/concurrent-visitors)     |Done       |
|[Typeahead suggestions for search](https://sys-design-interview.com/typeahead-suggestion)                 |Done       |
|[Online judge (like leetcode, hackerrank, codechef)](https://sys-design-interview.com/online-judge)       |Done       |
|[Metrics monitoring system](https://sys-design-interview.com/metrics-monitoring)                          |Done       |
|Proximity server/Yelp                                                                                     |In progress|
|TinyURL with multi-datacenter support                                                                     |Not started|
|Calendar                                                                                                  |Not started|
|Config push system                                                                                        |Not started|
|Zoom/Google Meet                                                                                          |Not started|
|FB messenger                                                                                              |Not started|
|P2P web crawler                                                                                           |Not started|
|Google Docs/Excalidraw                                                                                    |Not started|
|FB searching billions of posts                                                                            |Not started|
|Read, Copy, Update semantics                                                                              |Not started|



# Notes

* Calendar like Google, Outlook etc
  * [Extensible: All about recurrence](https://github.com/bmoeskau/Extensible/blob/master/recurrence-overview.md)
  * fanout problem, when groups can be added to the meetings.

* Config push system
  * developers can push configs via some client (eg, CLI)
  * consumer-libraries should listen to changes in configs
  * libraries can be running in several machines
  * config needs to be updated to all these consumer-libraries without requiring restarts
  * controlled rollout is preferred to avoid crashes and failures due to newly loaded config.

* Proximity server/Yelp
  * Main challenges is to index and search the locations
  * [This article](https://kousiknath.medium.com/system-design-design-a-geo-spatial-index-for-real-time-location-search-10968fe62b9c)
    has some useful details and other links.

* TinyURL with multi-datacenter support
  * 1000:1 read-to-write ratio
  * main challenges include replication scheme for datastore, caching strategy etc.

* Zoom/Google Meet
  * probably need to understand WebRTC, RTP, STUN, TURN server etc.

* P2P web crawler
  * [Chord: Distributed hash table](https://pdos.csail.mit.edu/papers/chord:sigcomm01/chord_sigcomm.pdf)
  * [Paper that proposes an architecture for the crawler](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.9.9637&rep=rep1&type=pdf)

* Google Docs/Excalidraw
  * architecture might look similar to the chat/FB messenger design
  * another main challenge is conflict resolution.
  * [Operational transformation visualization](https://operational-transformation.github.io/)

* FB searching billions of posts
  * Data volume is a challenge. Might need to restrict search to last ~6 months.

* RCU
  * Read, Copy, Update
  * reduces contention when there is a single writer (with relatively infrequent updates)
    and multiple readers (typically processing requests).
  * Youtube videos:
    * Basic intro to RCU: https://www.youtube.com/watch?v=rxQ5K9lo034
    * Talk by libGuarded author: https://www.youtube.com/watch?v=rNHLp44rMSs

<!-- Begin Mailchimp Signup Form -->
<link href="//cdn-images.mailchimp.com/embedcode/classic-10_7_dtp.css" rel="stylesheet" type="text/css">
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
	<div id="mce-responses" class="clear foot">
		<div class="response" id="mce-error-response" style="display:none"></div>
		<div class="response" id="mce-success-response" style="display:none"></div>
	</div>    <!-- real people should not fill this in and expect good things - do not remove this or risk form bot signups-->
    <div style="position: absolute; left: -5000px;" aria-hidden="true"><input type="text" name="b_37da8c24b585e6a4dd6785111_f35e3128e2" tabindex="-1" value=""></div>
        <div class="optionalParent">
            <div class="clear foot">
                <input type="submit" value="Subscribe" name="subscribe" id="mc-embedded-subscribe" class="button">
                <p class="brandingLogo"><a href="http://eepurl.com/hO6ax9" title="Mailchimp - email marketing made easy and fun"><img src="https://eep.io/mc-cdn-images/template_images/branding_logo_text_dark_dtp.svg"></a></p>
            </div>
        </div>
    </div>
</form>
</div>

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

