# Pagerank Project

In this project, you will create a simple search engine for the website <https://www.lawfareblog.com>.
This website provides legal analysis on US national security issues.

**Due date:** Sunday, 18 September at midnight

**Computation:**
This project has low computational requirements.
You are not required to complete it on the lambda server (although you are welcome to if you'd like).

## Background

**Data:**

The `data` folder contains two files that store example "web graphs".
The file `small.csv.gz` contains the example graph from the *Deeper Inside Pagerank* paper.
This is a small graph, so we can manually inspect the contents of this file with the following command:
```
$ zcat data/small.csv.gz
source,target
1,2
1,3
3,1
3,2
3,5
4,5
4,6
5,6
5,4
6,4
```

> **Recall:**
> The `cat` terminal command outputs the contents of a file to stdout, and the `zcat` command first decompressed a gzipped file and then outputs the decompressed contents.

As you can see, the graph is stored as a CSV file.
The first line is a header,
and each subsequent line stores a single edge in the graph.
The first column contains the source node of the edge and the second column the target node.
The file is assumed to be sorted alphabetically.

The second data file `lawfareblog.csv.gz` contains the link structure for the lawfare blog.
Let's take a look at the first 10 of these lines:
```
$ zcat data/lawfareblog.csv.gz | head
source,target
www.lawfareblog.com/,www.lawfareblog.com/topic/interrogation
www.lawfareblog.com/,www.lawfareblog.com/upcoming-events
www.lawfareblog.com/,www.lawfareblog.com/
www.lawfareblog.com/,www.lawfareblog.com/our-comments-policy
www.lawfareblog.com/,www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
www.lawfareblog.com/,www.lawfareblog.com/topic/lawfare-research-paper-series
www.lawfareblog.com/,www.lawfareblog.com/topic/book-reviews
www.lawfareblog.com/,www.lawfareblog.com/documents-related-mueller-investigation
www.lawfareblog.com/,www.lawfareblog.com/topic/international-law-loac
```
You can see that in this file, the node names are URLs.
Semantically, each line corresponds to an HTML `<a>` tag that is contained in the source webpage and links to the target webpage.

We can use the following command to count the total number of links in the file:
```
$ zcat data/lawfareblog.csv.gz | wc -l
1610789
```
Since every link corresponds to a non-zero entry in the `P` matrix,
this is also the value of `nnz(P)`.
(Technically, we should subtract 1 from this value since the `wc -l` command also counts the header line, not just the data lines.)

To get the dimensions of `P`, we need to count the total number of nodes in the graph.
The following command achieves this by: decompressing the file, extracting the first column, removing all duplicate lines, then counting the results.
```
$ zcat data/lawfareblog.csv.gz | cut -f1 -d, | uniq | wc -l
25761
```
This matrix is large enough that computing matrix products for dense matrices takes several minutes on a single CPU.
Fortunately, however, the matrix is very sparse.
The following python code computes the fraction of entries in the matrix with non-zero values:
```
>>> 1610788 / (25760**2)
0.0024274297384360172
```
Thus, by using sparse matrix operations, we will be able to speed up the code significantly.

**Code:**

The `pagerank.py` file contains code for loading the graph CSV files and searching through their nodes for key phrases.
For example, you can perform a search for all nodes (i.e. urls) that mention the string `corona` with the following command:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --search_query=corona
```

> **NOTE:**
> It will take about 10 seconds to load and parse the data files.
> All the other computation happens essentially instantly.

Currently, the pagerank of the nodes is not currently being calculated correctly, and so the webpages are returned in an arbitrary order.
Your task in this assignment will be to fix these calculations in order to have the most important results (i.e. highest pagerank results) returned first.

## Task 1: the power method

Implement the `WebGraph.power_method` function in `pagerank.py` for computing the pagerank vector by fixing the `FIXME` annotation.

**Part 1:**

To check that your implementation is working,
you should run the program on the `data/small.csv.gz` graph.
For my implementation, I get the following output.
```
$ python3 pagerank.py --data=data/small.csv.gz --verbose
DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 residual=2.5629e-01
DEBUG:root:i=1 residual=1.1841e-01
DEBUG:root:i=2 residual=7.0701e-02
DEBUG:root:i=3 residual=3.1815e-02
DEBUG:root:i=4 residual=2.0497e-02
DEBUG:root:i=5 residual=1.0108e-02
DEBUG:root:i=6 residual=6.3716e-03
DEBUG:root:i=7 residual=3.4228e-03
DEBUG:root:i=8 residual=2.0879e-03
DEBUG:root:i=9 residual=1.1750e-03
DEBUG:root:i=10 residual=7.0131e-04
DEBUG:root:i=11 residual=4.0321e-04
DEBUG:root:i=12 residual=2.3800e-04
DEBUG:root:i=13 residual=1.3812e-04
DEBUG:root:i=14 residual=8.1083e-05
DEBUG:root:i=15 residual=4.7251e-05
DEBUG:root:i=16 residual=2.7704e-05
DEBUG:root:i=17 residual=1.6164e-05
DEBUG:root:i=18 residual=9.4778e-06
DEBUG:root:i=19 residual=5.5066e-06
DEBUG:root:i=20 residual=3.2042e-06
DEBUG:root:i=21 residual=1.8612e-06
DEBUG:root:i=22 residual=1.1283e-06
DEBUG:root:i=23 residual=6.1907e-07
INFO:root:rank=0 pagerank=6.6270e-01 url=4
INFO:root:rank=1 pagerank=5.2179e-01 url=6
INFO:root:rank=2 pagerank=4.1434e-01 url=5
INFO:root:rank=3 pagerank=2.3175e-01 url=2
INFO:root:rank=4 pagerank=1.8590e-01 url=3
INFO:root:rank=5 pagerank=1.6917e-01 url=1
```
Yours likely won't be identical (due to weird floating point issues), but it should be similar.
In particular, the ranking of the nodes/urls should be the same order.

> **NOTE:**
> The `--verbose` flag causes all of the lines beginning with `DEBUG` to be printed.
> By default, only lines beginning with `INFO` are printed.

**Part 2:**

The `pagerank.py` file has an option `--search_query`, which takes a string as a parameter.
If this argument is used, then the program returns all nodes that match the query string sorted according to their pagerank.
Essentially, this gives us the most important pages related to our query.

Again, you may not get the exact same results as me,
but you should get similar results to the examples I've shown below.
Verify that you do in fact get similar results.

```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='corona'
INFO:root:rank=0 pagerank=1.0038e-03 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
INFO:root:rank=1 pagerank=8.9224e-04 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
INFO:root:rank=2 pagerank=7.0390e-04 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=3 pagerank=6.9153e-04 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=4 pagerank=6.7041e-04 url=www.lawfareblog.com/israeli-emergency-regulations-location-tracking-coronavirus-carriers
INFO:root:rank=5 pagerank=6.6256e-04 url=www.lawfareblog.com/why-congress-conducting-business-usual-face-coronavirus
INFO:root:rank=6 pagerank=6.5046e-04 url=www.lawfareblog.com/congressional-homeland-security-committees-seek-ways-support-state-federal-responses-coronavirus
INFO:root:rank=7 pagerank=6.3620e-04 url=www.lawfareblog.com/paper-hearing-experts-debate-digital-contact-tracing-and-coronavirus-privacy-concerns
INFO:root:rank=8 pagerank=6.1248e-04 url=www.lawfareblog.com/house-subcommittee-voices-concerns-over-us-management-coronavirus
INFO:root:rank=9 pagerank=6.0187e-04 url=www.lawfareblog.com/livestream-house-oversight-committee-holds-hearing-government-coronavirus-response

$ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='trump'
INFO:root:rank=0 pagerank=5.7826e-03 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=5.2338e-03 url=www.lawfareblog.com/document-trump-revokes-obama-executive-order-counterterrorism-strike-casualty-reporting
INFO:root:rank=2 pagerank=5.1297e-03 url=www.lawfareblog.com/trump-administrations-worrying-new-policy-israeli-settlements
INFO:root:rank=3 pagerank=4.6599e-03 url=www.lawfareblog.com/dc-circuit-overrules-district-courts-due-process-ruling-qasim-v-trump
INFO:root:rank=4 pagerank=4.5934e-03 url=www.lawfareblog.com/donald-trump-and-politically-weaponized-executive-branch
INFO:root:rank=5 pagerank=4.3071e-03 url=www.lawfareblog.com/how-trumps-approach-middle-east-ignores-past-future-and-human-condition
INFO:root:rank=6 pagerank=4.0935e-03 url=www.lawfareblog.com/why-trump-cant-buy-greenland
INFO:root:rank=7 pagerank=3.7591e-03 url=www.lawfareblog.com/oral-argument-summary-qassim-v-trump
INFO:root:rank=8 pagerank=3.4509e-03 url=www.lawfareblog.com/dc-circuit-court-denies-trump-rehearing-mazars-case
INFO:root:rank=9 pagerank=3.4484e-03 url=www.lawfareblog.com/second-circuit-rules-mazars-must-hand-over-trump-tax-returns-new-york-prosecutors

$ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='iran'
INFO:root:rank=0 pagerank=4.5746e-03 url=www.lawfareblog.com/praise-presidents-iran-tweets
INFO:root:rank=1 pagerank=4.4174e-03 url=www.lawfareblog.com/how-us-iran-tensions-could-disrupt-iraqs-fragile-peace
INFO:root:rank=2 pagerank=2.6928e-03 url=www.lawfareblog.com/cyber-command-operational-update-clarifying-june-2019-iran-operation
INFO:root:rank=3 pagerank=1.9391e-03 url=www.lawfareblog.com/aborted-iran-strike-fine-line-between-necessity-and-revenge
INFO:root:rank=4 pagerank=1.5452e-03 url=www.lawfareblog.com/parsing-state-departments-letter-use-force-against-iran
INFO:root:rank=5 pagerank=1.5357e-03 url=www.lawfareblog.com/iranian-hostage-crisis-and-its-effect-american-politics
INFO:root:rank=6 pagerank=1.5258e-03 url=www.lawfareblog.com/announcing-united-states-and-use-force-against-iran-new-lawfare-e-book
INFO:root:rank=7 pagerank=1.4221e-03 url=www.lawfareblog.com/us-names-iranian-revolutionary-guard-terrorist-organization-and-sanctions-international-criminal
INFO:root:rank=8 pagerank=1.1788e-03 url=www.lawfareblog.com/iran-shoots-down-us-drone-domestic-and-international-legal-implications
INFO:root:rank=9 pagerank=1.1463e-03 url=www.lawfareblog.com/israel-iran-syria-clash-and-law-use-force
```

**Part 3:**

The webgraph of lawfareblog.com (i.e. the `P` matrix) naturally contains a lot of structure.
For example, essentially all pages on the domain have links to the root page <https://lawfareblog.com/> and other "non-article" pages like <https://www.lawfareblog.com/topics> and <https://www.lawfareblog.com/subscribe-lawfare>.
These pages therefore have a large pagerank.
We can get a list of the pages with the largest pagerank by running

```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz
INFO:root:rank=0 pagerank=2.8741e-01 url=www.lawfareblog.com/lawfare-job-board
INFO:root:rank=1 pagerank=2.8741e-01 url=www.lawfareblog.com/masthead
INFO:root:rank=2 pagerank=2.8741e-01 url=www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
INFO:root:rank=3 pagerank=2.8741e-01 url=www.lawfareblog.com/documents-related-mueller-investigation
INFO:root:rank=4 pagerank=2.8741e-01 url=www.lawfareblog.com/topics
INFO:root:rank=5 pagerank=2.8741e-01 url=www.lawfareblog.com/about-lawfare-brief-history-term-and-site
INFO:root:rank=6 pagerank=2.8741e-01 url=www.lawfareblog.com/snowden-revelations
INFO:root:rank=7 pagerank=2.8741e-01 url=www.lawfareblog.com/support-lawfare
INFO:root:rank=8 pagerank=2.8741e-01 url=www.lawfareblog.com/upcoming-events
INFO:root:rank=9 pagerank=2.8741e-01 url=www.lawfareblog.com/our-comments-policy
```

Most of these pages are not very interesting, however, because they are not articles,
and usually when we are performing a web search, we only want articles.

This raises the question: How can we find the most important articles filtering out the non-article pages?
The answer is to modify the `P` matrix by removing all links to non-article pages.

This raises another question: How do we know if a link is a non-article page?
Unfortunately, this is a hard question to answer with 100% accuracy,
but there are many methods that get us most of the way there.
One easy to implement method is to compute what's called the "in-link ratio" of each node (i.e. the total number of edges with the node as a target divided by the total number of nodes),
and then remove nodes from the search results with too-high of a ratio.
The intuition is that non-article pages often appear in the menu of a webpage, and so have links from almost all of the other webpages;
but article-webpages are unlikely to appear on a menu and so will only have a small number of links from other webpages.
The `--filter_ratio` parameter causes the code to remove all pages that have an in-link ratio larger than the provided value.

Using this option, we can estimate the most important articles on the domain with the following command:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2
INFO:root:rank=0 pagerank=3.4696e-01 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=2.9521e-01 url=www.lawfareblog.com/livestream-nov-21-impeachment-hearings-0
INFO:root:rank=2 pagerank=2.9040e-01 url=www.lawfareblog.com/opening-statement-david-holmes
INFO:root:rank=3 pagerank=1.5179e-01 url=www.lawfareblog.com/lawfare-podcast-ben-nimmo-whack-mole-game-disinformation
INFO:root:rank=4 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1963
INFO:root:rank=5 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1964
INFO:root:rank=6 pagerank=1.5071e-01 url=www.lawfareblog.com/lawfare-podcast-week-was-impeachment
INFO:root:rank=7 pagerank=1.4957e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1962
INFO:root:rank=8 pagerank=1.4367e-01 url=www.lawfareblog.com/cyberlaw-podcast-mistrusting-google
INFO:root:rank=9 pagerank=1.4240e-01 url=www.lawfareblog.com/lawfare-podcast-bonus-edition-gordon-sondland-vs-committee-no-bull
```
Notice that the urls in this list look much more like articles than the urls in the previous list.

When Google calculates their `P` matrix for the web,
they use a similar (but much more complicated) process to modify the `P` matrix in order to reduce spam results.
The exact formula they use is a jealously guarded secret that they update continuously.

In the case above, notice that we have accidentally removed the blog's most popular article (<www.lawfareblog.com/snowden-revelations>).
The blog editors believed that Snowden's revelations about NSA spying are so important that they directly put a link to the article on the menu.
So every single webpage in the domain links to the Snowden article,
and our "anti-spam" `--filter-ratio` argument removed this article from the list.
In general, it is a challenging open problem to remove spam from pagerank results,
and all current solutions rely on careful human tuning and still have lots of false positives and false negatives.

**Part 4:**

Recall from the reading that the runtime of pagerank depends heavily on the eigengap of the `\bar\bar P` matrix,
and that this eigengap is bounded by the alpha parameter.

Run the following four commands:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose 
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --alpha=0.99999
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2 --alpha=0.99999
```
You should notice that the last command takes considerably more iterations to compute the pagerank vector.
(My code takes 685 iterations for this call, and about 10 iterations for all the others.)

This raises the question: Why does the second command (with the `--alpha` option but without the `--filter_ratio`) option not take a long time to run?
The answer is that the `P` graph for <https://www.lawfareblog.com> naturally has a large eigengap and so is fast to compute for all alpha values,
but the modified graph does not have a large eigengap and so requires a small alpha for fast convergence.

Changing the value of alpha also gives us very different pagerank rankings.
For example, 
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2
INFO:root:rank=0 pagerank=3.4696e-01 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=2.9521e-01 url=www.lawfareblog.com/livestream-nov-21-impeachment-hearings-0
INFO:root:rank=2 pagerank=2.9040e-01 url=www.lawfareblog.com/opening-statement-david-holmes
INFO:root:rank=3 pagerank=1.5179e-01 url=www.lawfareblog.com/lawfare-podcast-ben-nimmo-whack-mole-game-disinformation
INFO:root:rank=4 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1963
INFO:root:rank=5 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1964
INFO:root:rank=6 pagerank=1.5071e-01 url=www.lawfareblog.com/lawfare-podcast-week-was-impeachment
INFO:root:rank=7 pagerank=1.4957e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1962
INFO:root:rank=8 pagerank=1.4367e-01 url=www.lawfareblog.com/cyberlaw-podcast-mistrusting-google
INFO:root:rank=9 pagerank=1.4240e-01 url=www.lawfareblog.com/lawfare-podcast-bonus-edition-gordon-sondland-vs-committee-no-bull

$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2 --alpha=0.99999
INFO:root:rank=0 pagerank=7.0149e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=7.0149e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=1.0552e-01 url=www.lawfareblog.com/cost-using-zero-days
INFO:root:rank=3 pagerank=3.1755e-02 url=www.lawfareblog.com/lawfare-podcast-former-congressman-brian-baird-and-daniel-schuman-how-congress-can-continue-function
INFO:root:rank=4 pagerank=2.2040e-02 url=www.lawfareblog.com/events
INFO:root:rank=5 pagerank=1.6027e-02 url=www.lawfareblog.com/water-wars-increased-us-focus-indo-pacific
INFO:root:rank=6 pagerank=1.6026e-02 url=www.lawfareblog.com/water-wars-drill-maybe-drill
INFO:root:rank=7 pagerank=1.6023e-02 url=www.lawfareblog.com/water-wars-disjointed-operations-south-china-sea
INFO:root:rank=8 pagerank=1.6020e-02 url=www.lawfareblog.com/water-wars-song-oil-and-fire
INFO:root:rank=9 pagerank=1.6020e-02 url=www.lawfareblog.com/water-wars-sinking-feeling-philippine-china-relations
```

Which of these rankings is better is entirely subjective,
and the only way to know if you have the "best" alpha for your application is to try several variations and see what is best.
If large alphas are good for your application, you can see that there is a trade-off between quality answers and algorithmic runtime.
We'll be exploring this trade-off more formally in class over the rest of the semester.

## Task 2: the personalization vector

The most interesting applications of pagerank involve the personalization vector.
Implement the `WebGraph.make_personalization_vector` function so that it outputs a personalization vector tuned for the input query.
The pseudocode for the function is:
```
for each index in the personalization vector:
    get the url for the index (see the _index_to_url function)
    check if the url satisfies the input query (see the url_satisfies_query function)
    if so, set the corresponding index to one
normalize the vector
```

**Part 1:**

The command line argument `--personalization_vector_query` will use the function you created above to augment your search with a custom personalization vector.
If you've implemented the function correctly,
you should get results similar to:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona'
INFO:root:rank=0 pagerank=6.3127e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=6.3124e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=1.5947e-01 url=www.lawfareblog.com/chinatalk-how-party-takes-its-propaganda-global
INFO:root:rank=3 pagerank=1.2209e-01 url=www.lawfareblog.com/brexit-not-immune-coronavirus
INFO:root:rank=4 pagerank=1.2209e-01 url=www.lawfareblog.com/rational-security-my-corona-edition
INFO:root:rank=5 pagerank=9.3360e-02 url=www.lawfareblog.com/trump-cant-reopen-country-over-state-objections
INFO:root:rank=6 pagerank=9.1920e-02 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=7 pagerank=9.1920e-02 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=8 pagerank=7.7770e-02 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
INFO:root:rank=9 pagerank=7.2888e-02 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
```

Notice that these results are significantly different than when using the `--search_query` option:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --search_query='corona'
INFO:root:rank=0 pagerank=8.1320e-03 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
INFO:root:rank=1 pagerank=7.7908e-03 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
INFO:root:rank=2 pagerank=5.2262e-03 url=www.lawfareblog.com/livestream-house-oversight-committee-holds-hearing-government-coronavirus-response
INFO:root:rank=3 pagerank=3.9584e-03 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=4 pagerank=3.8114e-03 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=5 pagerank=3.3973e-03 url=www.lawfareblog.com/paper-hearing-experts-debate-digital-contact-tracing-and-coronavirus-privacy-concerns
INFO:root:rank=6 pagerank=3.3633e-03 url=www.lawfareblog.com/cyberlaw-podcast-how-israel-fighting-coronavirus
INFO:root:rank=7 pagerank=3.3557e-03 url=www.lawfareblog.com/israeli-emergency-regulations-location-tracking-coronavirus-carriers
INFO:root:rank=8 pagerank=3.2160e-03 url=www.lawfareblog.com/congress-needs-coronavirus-failsafe-its-too-late
INFO:root:rank=9 pagerank=3.1036e-03 url=www.lawfareblog.com/why-congress-conducting-business-usual-face-coronavirus
```

Which results are better?
Again, that depends on what you mean by "better."
With the `--personalization_vector_query` option,
a webpage is important only if other coronavirus webpages also think it's important;
with the `--search_query` option,
a webpage is important if any other webpage thinks it's important.
You'll notice that in the later example, many of the webpages are about Congressional proceedings related to the coronavirus.
From a strictly coronavirus perspective, these are not very important webpages.
But in the broader context of national security, these are very important webpages.

Google engineers spend TONs of time fine-tuning their pagerank personalization vectors to remove spam webpages.
Exactly how they do this is another one of their secrets that they don't publicly talk about.

**Part 2:**

Another use of the `--personalization_vector_query` option is that we can find out what webpages are related to the coronavirus but don't directly mention the coronavirus.
This can be used to map out what types of topics are similar to the coronavirus.

For example, the following query ranks all webpages by their `corona` importance,
but removes webpages mentioning `corona` from the results.
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona' --search_query='-corona'
INFO:root:rank=0 pagerank=6.3127e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=6.3124e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=1.5947e-01 url=www.lawfareblog.com/chinatalk-how-party-takes-its-propaganda-global
INFO:root:rank=3 pagerank=9.3360e-02 url=www.lawfareblog.com/trump-cant-reopen-country-over-state-objections
INFO:root:rank=4 pagerank=7.0277e-02 url=www.lawfareblog.com/fault-lines-foreign-policy-quarantined
INFO:root:rank=5 pagerank=6.9713e-02 url=www.lawfareblog.com/lawfare-podcast-mom-and-dad-talk-clinical-trials-pandemic
INFO:root:rank=6 pagerank=6.4944e-02 url=www.lawfareblog.com/limits-world-health-organization
INFO:root:rank=7 pagerank=5.9492e-02 url=www.lawfareblog.com/chinatalk-dispatches-shanghai-beijing-and-hong-kong
INFO:root:rank=8 pagerank=5.1245e-02 url=www.lawfareblog.com/us-moves-dismiss-case-against-company-linked-ira-troll-farm
INFO:root:rank=9 pagerank=5.1245e-02 url=www.lawfareblog.com/livestream-house-armed-services-holds-hearing-national-security-challenges-north-and-south-america
```
You can see that there are many urls about concepts that are obviously related like "covid", "clinical trials", and "quarantine",
but this algorithm also finds articles about Chinese propaganda and Trump's policy decisions.
Both of these articles are highly relevant to coronavirus discussions,
but a simple keyword search for corona or related terms would not find these articles.

<!--
**Part 3:**

Select another topic related to national security.
You should experiment with a national security topic other than the coronavirus.
For example, find out what articles are important to the `iran` topic but do not contain the word `iran`.
Your goal should be to discover what topics that www.lawfareblog.com considers to be related to the national security topic you choose.
-->

## Submission

1. Create a new repo on github (not a fork of this repo).

1. Run the following commands, and paste their output into the code blocks below.
   
   Task 1, part 1:
   ```
   $ python3 pagerank.py --data=data/small.csv.gz --verbose
        DEBUG:root:computing indices
        DEBUG:root:computing values
        n= 6
        DEBUG:root:i=0 residual=0.2562914192676544
        DEBUG:root:i=1 residual=0.11841227114200592
        DEBUG:root:i=2 residual=0.07070134580135345
        DEBUG:root:i=3 residual=0.03181542828679085
        DEBUG:root:i=4 residual=0.020496590062975883
        DEBUG:root:i=5 residual=0.01010835450142622
        DEBUG:root:i=6 residual=0.006371544674038887
        DEBUG:root:i=7 residual=0.0034227892756462097
        DEBUG:root:i=8 residual=0.002087961183860898
        DEBUG:root:i=9 residual=0.0011749734403565526
        DEBUG:root:i=10 residual=0.0007012754795141518
        DEBUG:root:i=11 residual=0.00040320929838344455
        DEBUG:root:i=12 residual=0.00023798426263965666
        DEBUG:root:i=13 residual=0.00013812065299134701
        DEBUG:root:i=14 residual=8.108324982458726e-05
        DEBUG:root:i=15 residual=4.7269360948121175e-05
        DEBUG:root:i=16 residual=2.7704918466042727e-05
        DEBUG:root:i=17 residual=1.6170568414963782e-05
        DEBUG:root:i=18 residual=9.479118489252869e-06
        DEBUG:root:i=19 residual=5.4782999541203026e-06
        DEBUG:root:i=20 residual=3.2123323308042018e-06
        DEBUG:root:i=21 residual=1.8802053318722756e-06
        DEBUG:root:i=22 residual=1.1228398761886638e-06
        DEBUG:root:i=23 residual=6.322027275018627e-07
        INFO:root:rank = 0 ranking = 6.6270e-01 url= 4
        INFO:root:rank = 1 ranking = 5.2179e-01 url= 6
        INFO:root:rank = 2 ranking = 4.1434e-01 url= 5
        INFO:root:rank = 3 ranking = 2.3175e-01 url= 2
        INFO:root:rank = 4 ranking = 1.8590e-01 url= 3
        INFO:root:rank = 5 ranking = 1.6917e-01 url= 1


   ```

   Task 1, part 2:
   ```
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='corona'
        n= 16862
        INFO:root:rank = 0 ranking = 1.8834e-08 url= www.lawfareblog.com/ted-cruz-vs-section-230-misrepresenting-communications-decency-act
        INFO:root:rank = 1 ranking = 1.8139e-08 url= www.lawfareblog.com/guess-what-play-ted-cruz-read-filibuster
        INFO:root:rank = 2 ranking = 1.8086e-08 url= www.lawfareblog.com/publicity-stunt-postscript-senators-cruz-and-paul-propose-legislation-targeted-killing-domestic
        INFO:root:rank = 3 ranking = 2.1523e-10 url= www.lawfareblog.com/ap-story-short-term-detention-facilities-afghanistan-and-gambling-casablanca
        INFO:root:rank = 4 ranking = 0.0000e+00 url= www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
        INFO:root:rank = 5 ranking = 0.0000e+00 url= www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
        INFO:root:rank = 6 ranking = 0.0000e+00 url= www.lawfareblog.com/britains-coronavirus-response
        INFO:root:rank = 7 ranking = 0.0000e+00 url= www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
        INFO:root:rank = 8 ranking = 0.0000e+00 url= www.lawfareblog.com/israeli-emergency-regulations-location-tracking-coronavirus-carriers
        INFO:root:rank = 9 ranking = 0.0000e+00 url= www.lawfareblog.com/why-congress-conducting-business-usual-face-coronavirus


   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='trump'
        n= 16862
        INFO:root:rank = 0 ranking = 1.5450e-09 url= www.lawfareblog.com/donald-trump-and-politically-weaponized-executive-branch
        INFO:root:rank = 1 ranking = 5.1209e-10 url= www.lawfareblog.com/did-donald-trump-jr-admit-violating-computer-fraud-and-abuse-act
        INFO:root:rank = 2 ranking = 2.9097e-10 url= www.lawfareblog.com/documents-saifullah-paracha-v-donald-j-trump
        INFO:root:rank = 3 ranking = 2.1813e-10 url= www.lawfareblog.com/cta9-decides-al-nashiri-v-macdonald
        INFO:root:rank = 4 ranking = 1.9674e-10 url= www.lawfareblog.com/donald-trump-danger-our-national-security
        INFO:root:rank = 5 ranking = 1.8614e-10 url= www.lawfareblog.com/burden-donald-trump
        INFO:root:rank = 6 ranking = 1.8245e-10 url= www.lawfareblog.com/does-trump-want-lose-eo-battle-court-or-donald-mcgahn-simply-ineffectual-or-worse
        INFO:root:rank = 7 ranking = 1.8139e-10 url= www.lawfareblog.com/nashiri-v-macdonald-dismissed
        INFO:root:rank = 8 ranking = 1.8072e-10 url= www.lawfareblog.com/more-donald-trump-and-justice-department
        INFO:root:rank = 9 ranking = 1.8064e-10 url= www.lawfareblog.com/donald-trump-paul-manafort-and-pesky-witness-tampering-statute
        
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='iran'
        n= 16862
        INFO:root:rank = 0 ranking = 1.1082e-05 url= www.lawfareblog.com/update-military-commissions-continued-health-issues-recusal-motion-and-new-cell-al-iraqi
        INFO:root:rank = 1 ranking = 9.7579e-06 url= www.lawfareblog.com/how-us-iran-tensions-could-disrupt-iraqs-fragile-peace
        INFO:root:rank = 2 ranking = 7.5627e-06 url= www.lawfareblog.com/france-makes-play-try-foreign-fighters-iraq
        INFO:root:rank = 3 ranking = 4.9849e-06 url= www.lawfareblog.com/document-sens-kaine-and-young-introduce-bill-revoke-iraq-force-authorizations
        INFO:root:rank = 4 ranking = 4.4441e-06 url= www.lawfareblog.com/haftar-attacking-tripoli-us-needs-re-engage-libya
        INFO:root:rank = 5 ranking = 4.2060e-06 url= www.lawfareblog.com/its-not-only-iraq-and-syria
        INFO:root:rank = 6 ranking = 3.8283e-06 url= www.lawfareblog.com/assessing-aclu-habeas-petition-behalf-unnamed-us-citizen-held-enemy-combatant-iraq
        INFO:root:rank = 7 ranking = 3.7903e-06 url= www.lawfareblog.com/primer-can-trump-administration-transfer-american-citizen-enemy-combatant-iraqi-custody
        INFO:root:rank = 8 ranking = 2.7731e-06 url= www.lawfareblog.com/2002-iraq-aumf-almost-certainly-authorizes-president-use-force-today-iraq-and-might-authorize-use
        INFO:root:rank = 9 ranking = 2.3138e-06 url= www.lawfareblog.com/iraq-inquiry-committee-releases-report-criticizing-uk-role-iraq-war
   ```

   Task 1, part 3:
   ```
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz
        n= 16862
        INFO:root:rank = 0 ranking = 2.8741e-01 url= www.lawfareblog.com/lawfare-job-board
        INFO:root:rank = 1 ranking = 2.8741e-01 url= www.lawfareblog.com/masthead
        INFO:root:rank = 2 ranking = 2.8741e-01 url= www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
        INFO:root:rank = 3 ranking = 2.8741e-01 url= www.lawfareblog.com/documents-related-mueller-investigation
        INFO:root:rank = 4 ranking = 2.8741e-01 url= www.lawfareblog.com/topics
        INFO:root:rank = 5 ranking = 2.8741e-01 url= www.lawfareblog.com/about-lawfare-brief-history-term-and-site
        INFO:root:rank = 6 ranking = 2.8741e-01 url= www.lawfareblog.com/snowden-revelations
        INFO:root:rank = 7 ranking = 2.8741e-01 url= www.lawfareblog.com/support-lawfare
        INFO:root:rank = 8 ranking = 2.8741e-01 url= www.lawfareblog.com/upcoming-events
        INFO:root:rank = 9 ranking = 2.8741e-01 url= www.lawfareblog.com/our-comments-policy


   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2
        n= 16862
        INFO:root:rank = 0 ranking = 3.4696e-01 url= www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
        INFO:root:rank = 1 ranking = 2.9521e-01 url= www.lawfareblog.com/livestream-nov-21-impeachment-hearings-0
        INFO:root:rank = 2 ranking = 2.9040e-01 url= www.lawfareblog.com/opening-statement-david-holmes
        INFO:root:rank = 3 ranking = 1.5179e-01 url= www.lawfareblog.com/lawfare-podcast-ben-nimmo-whack-mole-game-disinformation
        INFO:root:rank = 4 ranking = 1.5099e-01 url= www.lawfareblog.com/todays-headlines-and-commentary-1963
        INFO:root:rank = 5 ranking = 1.5099e-01 url= www.lawfareblog.com/todays-headlines-and-commentary-1964
        INFO:root:rank = 6 ranking = 1.5071e-01 url= www.lawfareblog.com/lawfare-podcast-week-was-impeachment
        INFO:root:rank = 7 ranking = 1.4957e-01 url= www.lawfareblog.com/todays-headlines-and-commentary-1962
        INFO:root:rank = 8 ranking = 1.4367e-01 url= www.lawfareblog.com/cyberlaw-podcast-mistrusting-google
        INFO:root:rank = 9 ranking = 1.4240e-01 url= www.lawfareblog.com/lawfare-podcast-bonus-edition-gordon-sondland-vs-committee-no-bull

   ```

   Task 1, part 4:
   ```
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose 
        DEBUG:root:computing indices
        DEBUG:root:computing values
        n= 16862
        DEBUG:root:i=0 residual=1.3793821334838867
        DEBUG:root:i=1 residual=0.11642514914274216
        DEBUG:root:i=2 residual=0.07495073974132538
        DEBUG:root:i=3 residual=0.031712669879198074
        DEBUG:root:i=4 residual=0.01746140420436859
        DEBUG:root:i=5 residual=0.008529536426067352
        DEBUG:root:i=6 residual=0.0044392067939043045
        DEBUG:root:i=7 residual=0.002238823566585779
        DEBUG:root:i=8 residual=0.0011464565759524703
        DEBUG:root:i=9 residual=0.0005798051133751869
        DEBUG:root:i=10 residual=0.00029213540256023407
        DEBUG:root:i=11 residual=0.00014553092478308827
        DEBUG:root:i=12 residual=7.149828888941556e-05
        DEBUG:root:i=13 residual=3.433692472754046e-05
        DEBUG:root:i=14 residual=1.5638515833416022e-05
        DEBUG:root:i=15 residual=6.266389391385019e-06
        DEBUG:root:i=16 residual=2.8251236017240444e-06
        DEBUG:root:i=17 residual=1.3609912912215805e-06
        DEBUG:root:i=18 residual=4.311152963509812e-07
        INFO:root:rank = 0 ranking = 2.8741e-01 url= www.lawfareblog.com/lawfare-job-board
        INFO:root:rank = 1 ranking = 2.8741e-01 url= www.lawfareblog.com/masthead
        INFO:root:rank = 2 ranking = 2.8741e-01 url= www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
        INFO:root:rank = 3 ranking = 2.8741e-01 url= www.lawfareblog.com/documents-related-mueller-investigation
        INFO:root:rank = 4 ranking = 2.8741e-01 url= www.lawfareblog.com/topics
        INFO:root:rank = 5 ranking = 2.8741e-01 url= www.lawfareblog.com/about-lawfare-brief-history-term-and-site
        INFO:root:rank = 6 ranking = 2.8741e-01 url= www.lawfareblog.com/snowden-revelations
        INFO:root:rank = 7 ranking = 2.8741e-01 url= www.lawfareblog.com/support-lawfare
        INFO:root:rank = 8 ranking = 2.8741e-01 url= www.lawfareblog.com/upcoming-events
        INFO:root:rank = 9 ranking = 2.8741e-01 url= www.lawfareblog.com/our-comments-policy
   
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --alpha=0.99999DEBUG:root:computing indices
        DEBUG:root:computing values
        n= 16862
        DEBUG:root:i=0 residual=1.3845856189727783
        DEBUG:root:i=1 residual=0.07088156789541245
        DEBUG:root:i=2 residual=0.01882227510213852
        DEBUG:root:i=3 residual=0.006958262529224157
        DEBUG:root:i=4 residual=0.0027358194347471
        DEBUG:root:i=5 residual=0.0010345563059672713
        DEBUG:root:i=6 residual=0.0003774634387809783
        DEBUG:root:i=7 residual=0.00013533401943277568
        DEBUG:root:i=8 residual=4.8224112106254324e-05
        DEBUG:root:i=9 residual=1.7172435036627576e-05
        DEBUG:root:i=10 residual=6.118058081483468e-06
        DEBUG:root:i=11 residual=2.173422217310872e-06
        DEBUG:root:i=12 residual=7.82504116614291e-07
        INFO:root:rank = 0 ranking = 2.8859e-01 url= www.lawfareblog.com/lawfare-job-board
        INFO:root:rank = 1 ranking = 2.8859e-01 url= www.lawfareblog.com/masthead
        INFO:root:rank = 2 ranking = 2.8859e-01 url= www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
        INFO:root:rank = 3 ranking = 2.8859e-01 url= www.lawfareblog.com/documents-related-mueller-investigation
        INFO:root:rank = 4 ranking = 2.8859e-01 url= www.lawfareblog.com/topics
        INFO:root:rank = 5 ranking = 2.8859e-01 url= www.lawfareblog.com/about-lawfare-brief-history-term-and-site
        INFO:root:rank = 6 ranking = 2.8859e-01 url= www.lawfareblog.com/snowden-revelations
        INFO:root:rank = 7 ranking = 2.8859e-01 url= www.lawfareblog.com/support-lawfare
        INFO:root:rank = 8 ranking = 2.8859e-01 url= www.lawfareblog.com/upcoming-events
        INFO:root:rank = 9 ranking = 2.8859e-01 url= www.lawfareblog.com/our-comments-policy
        
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2
        DEBUG:root:computing indices
        DEBUG:root:computing values
        n= 16862
        DEBUG:root:i=0 residual=1.2609827518463135
        DEBUG:root:i=1 residual=0.49858558177948
        DEBUG:root:i=2 residual=0.13420072197914124
        DEBUG:root:i=3 residual=0.0692318007349968
        DEBUG:root:i=4 residual=0.023411516100168228
        DEBUG:root:i=5 residual=0.010188630782067776
        DEBUG:root:i=6 residual=0.004910613875836134
        DEBUG:root:i=7 residual=0.002279882086440921
        DEBUG:root:i=8 residual=0.0010739191202446818
        DEBUG:root:i=9 residual=0.0005249588284641504
        DEBUG:root:i=10 residual=0.00026969940518029034
        DEBUG:root:i=11 residual=0.00014575093518942595
        DEBUG:root:i=12 residual=8.241771865868941e-05
        DEBUG:root:i=13 residual=4.819605601369403e-05
        DEBUG:root:i=14 residual=2.881278851418756e-05
        DEBUG:root:i=15 residual=1.7407368432031944e-05
        DEBUG:root:i=16 residual=1.0556699635344557e-05
        DEBUG:root:i=17 residual=6.386945642589126e-06
        DEBUG:root:i=18 residual=3.835165898635751e-06
        DEBUG:root:i=19 residual=2.295828835485736e-06
        DEBUG:root:i=20 residual=1.3655146631208481e-06
        DEBUG:root:i=21 residual=8.102273341137334e-07
        INFO:root:rank = 0 ranking = 3.4696e-01 url= www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
        INFO:root:rank = 1 ranking = 2.9521e-01 url= www.lawfareblog.com/livestream-nov-21-impeachment-hearings-0
        INFO:root:rank = 2 ranking = 2.9040e-01 url= www.lawfareblog.com/opening-statement-david-holmes
        INFO:root:rank = 3 ranking = 1.5179e-01 url= www.lawfareblog.com/lawfare-podcast-ben-nimmo-whack-mole-game-disinformation
        INFO:root:rank = 4 ranking = 1.5099e-01 url= www.lawfareblog.com/todays-headlines-and-commentary-1963
        INFO:root:rank = 5 ranking = 1.5099e-01 url= www.lawfareblog.com/todays-headlines-and-commentary-1964
        INFO:root:rank = 6 ranking = 1.5071e-01 url= www.lawfareblog.com/lawfare-podcast-week-was-impeachment
        INFO:root:rank = 7 ranking = 1.4957e-01 url= www.lawfareblog.com/todays-headlines-and-commentary-1962
        INFO:root:rank = 8 ranking = 1.4367e-01 url= www.lawfareblog.com/cyberlaw-podcast-mistrusting-google
        INFO:root:rank = 9 ranking = 1.4240e-01 url= www.lawfareblog.com/lawfare-podcast-bonus-edition-gordon-sondland-vs-committee-no-bull

        
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2 --alpha=0.99999
        DEBUG:root:computing indices
        DEBUG:root:computing values
        n= 16862
        DEBUG:root:i=0 residual=1.2827345132827759
        DEBUG:root:i=1 residual=0.5695679783821106
        DEBUG:root:i=2 residual=0.38298743963241577
        DEBUG:root:i=3 residual=0.2173907607793808
        DEBUG:root:i=4 residual=0.14045004546642303
        DEBUG:root:i=5 residual=0.10851320624351501
        DEBUG:root:i=6 residual=0.09284130483865738
        DEBUG:root:i=7 residual=0.08225572854280472
        DEBUG:root:i=8 residual=0.07338864356279373
        DEBUG:root:i=9 residual=0.06561222672462463
        DEBUG:root:i=10 residual=0.059096433222293854
        DEBUG:root:i=11 residual=0.0541754886507988
        DEBUG:root:i=12 residual=0.05111689865589142
        DEBUG:root:i=13 residual=0.04999383166432381
        DEBUG:root:i=14 residual=0.05060902237892151
        DEBUG:root:i=15 residual=0.0525263175368309
        DEBUG:root:i=16 residual=0.0551888681948185
        DEBUG:root:i=17 residual=0.05803852900862694
        DEBUG:root:i=18 residual=0.06059221178293228
        DEBUG:root:i=19 residual=0.0624784380197525
        DEBUG:root:i=20 residual=0.06345319747924805
        DEBUG:root:i=21 residual=0.06340522319078445
        DEBUG:root:i=22 residual=0.06234566867351532
        DEBUG:root:i=23 residual=0.06038395315408707
        DEBUG:root:i=24 residual=0.05769402161240578
        DEBUG:root:i=25 residual=0.05447976291179657
        DEBUG:root:i=26 residual=0.05094277113676071
        DEBUG:root:i=27 residual=0.0472610704600811
        DEBUG:root:i=28 residual=0.043578535318374634
        DEBUG:root:i=29 residual=0.04000161215662956
        DEBUG:root:i=30 residual=0.03660248592495918
        DEBUG:root:i=31 residual=0.033424295485019684
        DEBUG:root:i=32 residual=0.030489472672343254
        DEBUG:root:i=33 residual=0.02780323475599289
        DEBUG:root:i=34 residual=0.025360697880387306
        DEBUG:root:i=35 residual=0.023150015622377396
        DEBUG:root:i=36 residual=0.02115531638264656
        DEBUG:root:i=37 residual=0.019359197467565536
        DEBUG:root:i=38 residual=0.01774349994957447
        DEBUG:root:i=39 residual=0.016290761530399323
        DEBUG:root:i=40 residual=0.0149843106046319
        DEBUG:root:i=41 residual=0.01380856242030859
        DEBUG:root:i=42 residual=0.012749706394970417
        DEBUG:root:i=43 residual=0.011794854886829853
        DEBUG:root:i=44 residual=0.010932852514088154
        DEBUG:root:i=45 residual=0.010153341107070446
        DEBUG:root:i=46 residual=0.009447379969060421
        DEBUG:root:i=47 residual=0.008807037957012653
        DEBUG:root:i=48 residual=0.00822509080171585
        DEBUG:root:i=49 residual=0.007695489563047886
        DEBUG:root:i=50 residual=0.007212585769593716
        DEBUG:root:i=51 residual=0.00677154865115881
        DEBUG:root:i=52 residual=0.006367980502545834
        DEBUG:root:i=53 residual=0.005998112726956606
        DEBUG:root:i=54 residual=0.005658607929944992
        DEBUG:root:i=55 residual=0.005346357356756926
        DEBUG:root:i=56 residual=0.005058770999312401
        DEBUG:root:i=57 residual=0.004793448373675346
        DEBUG:root:i=58 residual=0.004548261873424053
        DEBUG:root:i=59 residual=0.004321318585425615
        DEBUG:root:i=60 residual=0.004110995680093765
        DEBUG:root:i=61 residual=0.003915745764970779
        DEBUG:root:i=62 residual=0.0037342202849686146
        DEBUG:root:i=63 residual=0.0035651903599500656
        DEBUG:root:i=64 residual=0.0034076401498168707
        DEBUG:root:i=65 residual=0.0032605668529868126
        DEBUG:root:i=66 residual=0.0031230042222887278
        DEBUG:root:i=67 residual=0.0029942821711301804
        DEBUG:root:i=68 residual=0.0028736344538629055
        DEBUG:root:i=69 residual=0.002760394709184766
        DEBUG:root:i=70 residual=0.0026539983227849007
        DEBUG:root:i=71 residual=0.002553925383836031
        DEBUG:root:i=72 residual=0.0024596548173576593
        DEBUG:root:i=73 residual=0.0023707826621830463
        DEBUG:root:i=74 residual=0.0022869009990245104
        DEBUG:root:i=75 residual=0.0022076410241425037
        DEBUG:root:i=76 residual=0.0021326374262571335
        DEBUG:root:i=77 residual=0.0020616468973457813
        DEBUG:root:i=78 residual=0.0019943546503782272
        DEBUG:root:i=79 residual=0.001930529368110001
        DEBUG:root:i=80 residual=0.0018698793137446046
        DEBUG:root:i=81 residual=0.0018122568726539612
        DEBUG:root:i=82 residual=0.0017574297962710261
        DEBUG:root:i=83 residual=0.001705214031971991
        DEBUG:root:i=84 residual=0.0016554755857214332
        DEBUG:root:i=85 residual=0.0016080320347100496
        DEBUG:root:i=86 residual=0.0015627194661647081
        DEBUG:root:i=87 residual=0.001519430079497397
        DEBUG:root:i=88 residual=0.0014780594501644373
        DEBUG:root:i=89 residual=0.0014384739333763719
        DEBUG:root:i=90 residual=0.0014005572302266955
        DEBUG:root:i=91 residual=0.0013642337871715426
        DEBUG:root:i=92 residual=0.0013293965021148324
        DEBUG:root:i=93 residual=0.0012959778541699052
        DEBUG:root:i=94 residual=0.001263876329176128
        DEBUG:root:i=95 residual=0.0012330460594967008
        DEBUG:root:i=96 residual=0.0012033917009830475
        DEBUG:root:i=97 residual=0.001174872275441885
        DEBUG:root:i=98 residual=0.0011474145576357841
        DEBUG:root:i=99 residual=0.0011209737276658416
        DEBUG:root:i=100 residual=0.0010954836616292596
        DEBUG:root:i=101 residual=0.0010709064081311226
        DEBUG:root:i=102 residual=0.0010471936548128724
        DEBUG:root:i=103 residual=0.0010242975549772382
        DEBUG:root:i=104 residual=0.00100218434818089
        DEBUG:root:i=105 residual=0.00098081910982728
        DEBUG:root:i=106 residual=0.0009601557976566255
        DEBUG:root:i=107 residual=0.0009401643765158951
        DEBUG:root:i=108 residual=0.0009208175470121205
        DEBUG:root:i=109 residual=0.0009020979050546885
        DEBUG:root:i=110 residual=0.0008839573129080236
        DEBUG:root:i=111 residual=0.000866368121933192
        DEBUG:root:i=112 residual=0.0008493199711665511
        DEBUG:root:i=113 residual=0.0008327721152454615
        DEBUG:root:i=114 residual=0.000816727289929986
        DEBUG:root:i=115 residual=0.0008011464960873127
        DEBUG:root:i=116 residual=0.0007860094192437828
        DEBUG:root:i=117 residual=0.0007713042432442307
        DEBUG:root:i=118 residual=0.0007570049492642283
        DEBUG:root:i=119 residual=0.0007431192207150161
        DEBUG:root:i=120 residual=0.0007295997929759324
        DEBUG:root:i=121 residual=0.0007164512644521892
        DEBUG:root:i=122 residual=0.0007036454626359046
        DEBUG:root:i=123 residual=0.0006911859381943941
        DEBUG:root:i=124 residual=0.0006790339248254895
        DEBUG:root:i=125 residual=0.0006672092131339014
        DEBUG:root:i=126 residual=0.0006556769367307425
        DEBUG:root:i=127 residual=0.000644430925603956
        DEBUG:root:i=128 residual=0.0006334679783321917
        DEBUG:root:i=129 residual=0.0006227655103430152
        DEBUG:root:i=130 residual=0.0006123287603259087
        DEBUG:root:i=131 residual=0.0006021387525834143
        DEBUG:root:i=132 residual=0.0005921924021095037
        DEBUG:root:i=133 residual=0.0005824716645292938
        DEBUG:root:i=134 residual=0.0005729826516471803
        DEBUG:root:i=135 residual=0.0005637070862576365
        DEBUG:root:i=136 residual=0.0005546381580643356
        DEBUG:root:i=137 residual=0.000545780174434185
        DEBUG:root:i=138 residual=0.0005371167790144682
        DEBUG:root:i=139 residual=0.0005286391242407262
        DEBUG:root:i=140 residual=0.0005203530308790505
        DEBUG:root:i=141 residual=0.0005122329457663
        DEBUG:root:i=142 residual=0.0005042985430918634
        DEBUG:root:i=143 residual=0.0004965272382833064
        DEBUG:root:i=144 residual=0.0004889093688689172
        DEBUG:root:i=145 residual=0.00048145302571356297
        DEBUG:root:i=146 residual=0.0004741534066852182
        DEBUG:root:i=147 residual=0.00046700096572749317
        DEBUG:root:i=148 residual=0.0004599933745339513
        DEBUG:root:i=149 residual=0.00045312679139897227
        DEBUG:root:i=150 residual=0.0004463912337087095
        DEBUG:root:i=151 residual=0.00043978646863251925
        DEBUG:root:i=152 residual=0.00043331467895768583
        DEBUG:root:i=153 residual=0.00042697106255218387
        DEBUG:root:i=154 residual=0.0004207405145280063
        DEBUG:root:i=155 residual=0.00041463071829639375
        DEBUG:root:i=156 residual=0.0004086443514097482
        DEBUG:root:i=157 residual=0.00040276054642163217
        DEBUG:root:i=158 residual=0.00039698302862234414
        DEBUG:root:i=159 residual=0.0003913186374120414
        DEBUG:root:i=160 residual=0.0003857545671053231
        DEBUG:root:i=161 residual=0.00038029320421628654
        DEBUG:root:i=162 residual=0.0003749268944375217
        DEBUG:root:i=163 residual=0.00036965610343031585
        DEBUG:root:i=164 residual=0.0003644881653599441
        DEBUG:root:i=165 residual=0.0003593963338062167
        DEBUG:root:i=166 residual=0.00035439812927506864
        DEBUG:root:i=167 residual=0.00034949349355883896
        DEBUG:root:i=168 residual=0.00034466685610823333
        DEBUG:root:i=169 residual=0.00033993145916610956
        DEBUG:root:i=170 residual=0.00033526966581121087
        DEBUG:root:i=171 residual=0.0003306938160676509
        DEBUG:root:i=172 residual=0.00032618685509078205
        DEBUG:root:i=173 residual=0.00032175728119909763
        DEBUG:root:i=174 residual=0.000317403202643618
        DEBUG:root:i=175 residual=0.00031312249484471977
        DEBUG:root:i=176 residual=0.0003089130623266101
        DEBUG:root:i=177 residual=0.00030476629035547376
        DEBUG:root:i=178 residual=0.00030069483909755945
        DEBUG:root:i=179 residual=0.00029668453498743474
        DEBUG:root:i=180 residual=0.0002927415771409869
        DEBUG:root:i=181 residual=0.0002888553717639297
        DEBUG:root:i=182 residual=0.0002850376768037677
        DEBUG:root:i=183 residual=0.0002812762977555394
        DEBUG:root:i=184 residual=0.0002775781322270632
        DEBUG:root:i=185 residual=0.0002739399787969887
        DEBUG:root:i=186 residual=0.0002703559584915638
        DEBUG:root:i=187 residual=0.0002668243832886219
        DEBUG:root:i=188 residual=0.0002633541589602828
        DEBUG:root:i=189 residual=0.00025993023882620037
        DEBUG:root:i=190 residual=0.0002565619070082903
        DEBUG:root:i=191 residual=0.00025324535090476274
        DEBUG:root:i=192 residual=0.0002499781840015203
        DEBUG:root:i=193 residual=0.0002467619488015771
        DEBUG:root:i=194 residual=0.00024359324015676975
        DEBUG:root:i=195 residual=0.00024047000624705106
        DEBUG:root:i=196 residual=0.00023739771859254688
        DEBUG:root:i=197 residual=0.00023436793708242476
        DEBUG:root:i=198 residual=0.00023138622054830194
        DEBUG:root:i=199 residual=0.00022844658815301955
        DEBUG:root:i=200 residual=0.0002255516592413187
        DEBUG:root:i=201 residual=0.00022269775217864662
        DEBUG:root:i=202 residual=0.00021988390653859824
        DEBUG:root:i=203 residual=0.00021711080626118928
        DEBUG:root:i=204 residual=0.00021437922259792686
        DEBUG:root:i=205 residual=0.000211689795833081
        DEBUG:root:i=206 residual=0.00020903782569803298
        DEBUG:root:i=207 residual=0.00020642278832383454
        DEBUG:root:i=208 residual=0.00020384354866109788
        DEBUG:root:i=209 residual=0.00020130380289629102
        DEBUG:root:i=210 residual=0.00019879733736161143
        DEBUG:root:i=211 residual=0.0001963248651009053
        DEBUG:root:i=212 residual=0.00019388859800528735
        DEBUG:root:i=213 residual=0.0001914903405122459
        DEBUG:root:i=214 residual=0.00018912261293735355
        DEBUG:root:i=215 residual=0.00018678774358704686
        DEBUG:root:i=216 residual=0.0001844896760303527
        DEBUG:root:i=217 residual=0.00018221828213427216
        DEBUG:root:i=218 residual=0.00017998278781305999
        DEBUG:root:i=219 residual=0.00017777625180315226
        DEBUG:root:i=220 residual=0.00017559819389134645
        DEBUG:root:i=221 residual=0.00017345235391985625
        DEBUG:root:i=222 residual=0.00017133599612861872
        DEBUG:root:i=223 residual=0.0001692451915005222
        DEBUG:root:i=224 residual=0.00016718772531021386
        DEBUG:root:i=225 residual=0.00016515505558345467
        DEBUG:root:i=226 residual=0.0001631500053917989
        DEBUG:root:i=227 residual=0.00016117207997012883
        DEBUG:root:i=228 residual=0.00015922266175039113
        DEBUG:root:i=229 residual=0.00015729584265500307
        DEBUG:root:i=230 residual=0.00015539934975095093
        DEBUG:root:i=231 residual=0.00015352752234321088
        DEBUG:root:i=232 residual=0.000151676926179789
        DEBUG:root:i=233 residual=0.00014985409507062286
        DEBUG:root:i=234 residual=0.000148051927681081
        DEBUG:root:i=235 residual=0.00014627586642745882
        DEBUG:root:i=236 residual=0.00014452151663135737
        DEBUG:root:i=237 residual=0.0001427954703103751
        DEBUG:root:i=238 residual=0.00014108797768130898
        DEBUG:root:i=239 residual=0.00013940119242761284
        DEBUG:root:i=240 residual=0.00013773904356639832
        DEBUG:root:i=241 residual=0.00013610039604827762
        DEBUG:root:i=242 residual=0.00013448078243527561
        DEBUG:root:i=243 residual=0.0001328849175479263
        DEBUG:root:i=244 residual=0.00013130728621035814
        DEBUG:root:i=245 residual=0.00012974858691450208
        DEBUG:root:i=246 residual=0.00012821114796679467
        DEBUG:root:i=247 residual=0.00012669485295191407
        DEBUG:root:i=248 residual=0.00012519625306595117
        DEBUG:root:i=249 residual=0.00012371878256089985
        DEBUG:root:i=250 residual=0.000122259501949884
        DEBUG:root:i=251 residual=0.00012081880413461477
        DEBUG:root:i=252 residual=0.00011939798423554748
        DEBUG:root:i=253 residual=0.00011799210187746212
        DEBUG:root:i=254 residual=0.00011660611198749393
        DEBUG:root:i=255 residual=0.00011523616558406502
        DEBUG:root:i=256 residual=0.00011388628627173603
        DEBUG:root:i=257 residual=0.00011255133722443134
        DEBUG:root:i=258 residual=0.00011123330477857962
        DEBUG:root:i=259 residual=0.0001099329165299423
        DEBUG:root:i=260 residual=0.0001086478732759133
        DEBUG:root:i=261 residual=0.00010738023411249742
        DEBUG:root:i=262 residual=0.00010612714686430991
        DEBUG:root:i=263 residual=0.00010488892439752817
        DEBUG:root:i=264 residual=0.00010366823698859662
        DEBUG:root:i=265 residual=0.00010246112651657313
        DEBUG:root:i=266 residual=0.00010126895358553156
        DEBUG:root:i=267 residual=0.00010009211837314069
        DEBUG:root:i=268 residual=9.892899106489494e-05
        DEBUG:root:i=269 residual=9.778246749192476e-05
        DEBUG:root:i=270 residual=9.665037941886112e-05
        DEBUG:root:i=271 residual=9.553277777740732e-05
        DEBUG:root:i=272 residual=9.442564623896033e-05
        DEBUG:root:i=273 residual=9.333469643024728e-05
        DEBUG:root:i=274 residual=9.225682151736692e-05
        DEBUG:root:i=275 residual=9.119172318605706e-05
        DEBUG:root:i=276 residual=9.013924864120781e-05
        DEBUG:root:i=277 residual=8.910078759072348e-05
        DEBUG:root:i=278 residual=8.807330596027896e-05
        DEBUG:root:i=279 residual=8.706083463039249e-05
        DEBUG:root:i=280 residual=8.605887705925852e-05
        DEBUG:root:i=281 residual=8.507017628289759e-05
        DEBUG:root:i=282 residual=8.409364090766758e-05
        DEBUG:root:i=283 residual=8.312944555655122e-05
        DEBUG:root:i=284 residual=8.217340655392036e-05
        DEBUG:root:i=285 residual=8.12315265648067e-05
        DEBUG:root:i=286 residual=8.03016300778836e-05
        DEBUG:root:i=287 residual=7.93828148744069e-05
        DEBUG:root:i=288 residual=7.847524102544412e-05
        DEBUG:root:i=289 residual=7.757759158266708e-05
        DEBUG:root:i=290 residual=7.669065962545574e-05
        DEBUG:root:i=291 residual=7.581511454191059e-05
        DEBUG:root:i=292 residual=7.495055615436286e-05
        DEBUG:root:i=293 residual=7.409635873045772e-05
        DEBUG:root:i=294 residual=7.325167825911194e-05
        DEBUG:root:i=295 residual=7.241672574309632e-05
        DEBUG:root:i=296 residual=7.159197411965579e-05
        DEBUG:root:i=297 residual=7.077823829604313e-05
        DEBUG:root:i=298 residual=6.997418677201495e-05
        DEBUG:root:i=299 residual=6.917937571415678e-05
        DEBUG:root:i=300 residual=6.839257548563182e-05
        DEBUG:root:i=301 residual=6.76158961141482e-05
        DEBUG:root:i=302 residual=6.68485663481988e-05
        DEBUG:root:i=303 residual=6.609157571801916e-05
        DEBUG:root:i=304 residual=6.534181738970801e-05
        DEBUG:root:i=305 residual=6.4601976191625e-05
        DEBUG:root:i=306 residual=6.387125904439017e-05
        DEBUG:root:i=307 residual=6.314684287644923e-05
        DEBUG:root:i=308 residual=6.24311578576453e-05
        DEBUG:root:i=309 residual=6.172778375912458e-05
        DEBUG:root:i=310 residual=6.102908810134977e-05
        DEBUG:root:i=311 residual=6.034163379808888e-05
        DEBUG:root:i=312 residual=5.9659447288140655e-05
        DEBUG:root:i=313 residual=5.8985835494240746e-05
        DEBUG:root:i=314 residual=5.832106762682088e-05
        DEBUG:root:i=315 residual=5.7661956816446036e-05
        DEBUG:root:i=316 residual=5.7010725868167356e-05
        DEBUG:root:i=317 residual=5.636864443658851e-05
        DEBUG:root:i=318 residual=5.573602902586572e-05
        DEBUG:root:i=319 residual=5.510967821464874e-05
        DEBUG:root:i=320 residual=5.4487798479385674e-05
        DEBUG:root:i=321 residual=5.387514102039859e-05
        DEBUG:root:i=322 residual=5.3271356591722e-05
        DEBUG:root:i=323 residual=5.2671115554403514e-05
        DEBUG:root:i=324 residual=5.2080988098168746e-05
        DEBUG:root:i=325 residual=5.1494589570211247e-05
        DEBUG:root:i=326 residual=5.091816638014279e-05
        DEBUG:root:i=327 residual=5.034670903114602e-05
        DEBUG:root:i=328 residual=4.978231663699262e-05
        DEBUG:root:i=329 residual=4.9225069233216345e-05
        DEBUG:root:i=330 residual=4.867307870881632e-05
        DEBUG:root:i=331 residual=4.8128153139259666e-05
        DEBUG:root:i=332 residual=4.7589681344106793e-05
        DEBUG:root:i=333 residual=4.705690298578702e-05
        DEBUG:root:i=334 residual=4.6531040425179526e-05
        DEBUG:root:i=335 residual=4.6010190999368206e-05
        DEBUG:root:i=336 residual=4.5495846279663965e-05
        DEBUG:root:i=337 residual=4.4988344598095864e-05
        DEBUG:root:i=338 residual=4.4486005208455026e-05
        DEBUG:root:i=339 residual=4.398934470373206e-05
        DEBUG:root:i=340 residual=4.349655864643864e-05
        DEBUG:root:i=341 residual=4.3013409595005214e-05
        DEBUG:root:i=342 residual=4.253243969287723e-05
        DEBUG:root:i=343 residual=4.205781806376763e-05
        DEBUG:root:i=344 residual=4.15892900491599e-05
        DEBUG:root:i=345 residual=4.1126848373096436e-05
        DEBUG:root:i=346 residual=4.06678591389209e-05
        DEBUG:root:i=347 residual=4.0215432818513364e-05
        DEBUG:root:i=348 residual=3.9766906411387026e-05
        DEBUG:root:i=349 residual=3.932619438273832e-05
        DEBUG:root:i=350 residual=3.888847277266905e-05
        DEBUG:root:i=351 residual=3.845540777547285e-05
        DEBUG:root:i=352 residual=3.8027792470529675e-05
        DEBUG:root:i=353 residual=3.7605612305924296e-05
        DEBUG:root:i=354 residual=3.71875285054557e-05
        DEBUG:root:i=355 residual=3.6773577448911965e-05
        DEBUG:root:i=356 residual=3.636644396465272e-05
        DEBUG:root:i=357 residual=3.596319584175944e-05
        DEBUG:root:i=358 residual=3.556347655830905e-05
        DEBUG:root:i=359 residual=3.516770084388554e-05
        DEBUG:root:i=360 residual=3.477870632195845e-05
        DEBUG:root:i=361 residual=3.439400461502373e-05
        DEBUG:root:i=362 residual=3.401244612177834e-05
        DEBUG:root:i=363 residual=3.363522046129219e-05
        DEBUG:root:i=364 residual=3.3263237128267065e-05
        DEBUG:root:i=365 residual=3.289396772743203e-05
        DEBUG:root:i=366 residual=3.2530784665141255e-05
        DEBUG:root:i=367 residual=3.2169988116947934e-05
        DEBUG:root:i=368 residual=3.181412466801703e-05
        DEBUG:root:i=369 residual=3.1461087928619236e-05
        DEBUG:root:i=370 residual=3.1113737350096926e-05
        DEBUG:root:i=371 residual=3.076847497140989e-05
        DEBUG:root:i=372 residual=3.0428151148953475e-05
        DEBUG:root:i=373 residual=3.009144165844191e-05
        DEBUG:root:i=374 residual=2.975965617224574e-05
        DEBUG:root:i=375 residual=2.94317833322566e-05
        DEBUG:root:i=376 residual=2.9105282010277733e-05
        DEBUG:root:i=377 residual=2.87856728391489e-05
        DEBUG:root:i=378 residual=2.8467313313740306e-05
        DEBUG:root:i=379 residual=2.8152218874311075e-05
        DEBUG:root:i=380 residual=2.7842044801218435e-05
        DEBUG:root:i=381 residual=2.75348993454827e-05
        DEBUG:root:i=382 residual=2.7230718842474744e-05
        DEBUG:root:i=383 residual=2.6931273168884218e-05
        DEBUG:root:i=384 residual=2.663364284671843e-05
        DEBUG:root:i=385 residual=2.6341316697653383e-05
        DEBUG:root:i=386 residual=2.6050645828945562e-05
        DEBUG:root:i=387 residual=2.5763265512068756e-05
        DEBUG:root:i=388 residual=2.5478786483290605e-05
        DEBUG:root:i=389 residual=2.5198274670401588e-05
        DEBUG:root:i=390 residual=2.4922792363213375e-05
        DEBUG:root:i=391 residual=2.464870340190828e-05
        DEBUG:root:i=392 residual=2.437617331452202e-05
        DEBUG:root:i=393 residual=2.4108247089316137e-05
        DEBUG:root:i=394 residual=2.3842987502575852e-05
        DEBUG:root:i=395 residual=2.3580560082336888e-05
        DEBUG:root:i=396 residual=2.332206531718839e-05
        DEBUG:root:i=397 residual=2.3064507331582718e-05
        DEBUG:root:i=398 residual=2.281130764458794e-05
        DEBUG:root:i=399 residual=2.256016159662977e-05
        DEBUG:root:i=400 residual=2.2313877707347274e-05
        DEBUG:root:i=401 residual=2.2066791643737815e-05
        DEBUG:root:i=402 residual=2.1823971110279672e-05
        DEBUG:root:i=403 residual=2.1585472495644353e-05
        DEBUG:root:i=404 residual=2.1347377696656622e-05
        DEBUG:root:i=405 residual=2.111458161380142e-05
        DEBUG:root:i=406 residual=2.0882851458736695e-05
        DEBUG:root:i=407 residual=2.065289118036162e-05
        DEBUG:root:i=408 residual=2.042709274974186e-05
        DEBUG:root:i=409 residual=2.02011924557155e-05
        DEBUG:root:i=410 residual=1.9979521312052384e-05
        DEBUG:root:i=411 residual=1.9761322619160637e-05
        DEBUG:root:i=412 residual=1.954420076799579e-05
        DEBUG:root:i=413 residual=1.9329343558638357e-05
        DEBUG:root:i=414 residual=1.9118924683425575e-05
        DEBUG:root:i=415 residual=1.8909277059719898e-05
        DEBUG:root:i=416 residual=1.8700478904065676e-05
        DEBUG:root:i=417 residual=1.849528052844107e-05
        DEBUG:root:i=418 residual=1.829484426707495e-05
        DEBUG:root:i=419 residual=1.809328568924684e-05
        DEBUG:root:i=420 residual=1.7895181372296065e-05
        DEBUG:root:i=421 residual=1.7699003365123644e-05
        DEBUG:root:i=422 residual=1.7504647985333577e-05
        DEBUG:root:i=423 residual=1.731540760374628e-05
        DEBUG:root:i=424 residual=1.7124788428191096e-05
        DEBUG:root:i=425 residual=1.6935324310907163e-05
        DEBUG:root:i=426 residual=1.675023668212816e-05
        DEBUG:root:i=427 residual=1.6569147192058153e-05
        DEBUG:root:i=428 residual=1.638637695577927e-05
        DEBUG:root:i=429 residual=1.620819602976553e-05
        DEBUG:root:i=430 residual=1.602952579560224e-05
        DEBUG:root:i=431 residual=1.5855599485803396e-05
        DEBUG:root:i=432 residual=1.568147672514897e-05
        DEBUG:root:i=433 residual=1.5508729120483622e-05
        DEBUG:root:i=434 residual=1.5339463061536662e-05
        DEBUG:root:i=435 residual=1.517303280706983e-05
        DEBUG:root:i=436 residual=1.5007686670287512e-05
        DEBUG:root:i=437 residual=1.484225140302442e-05
        DEBUG:root:i=438 residual=1.4680577805847861e-05
        DEBUG:root:i=439 residual=1.4519720025418792e-05
        DEBUG:root:i=440 residual=1.4362240108312108e-05
        DEBUG:root:i=441 residual=1.4204974831955042e-05
        DEBUG:root:i=442 residual=1.4047832337382715e-05
        DEBUG:root:i=443 residual=1.3894668882130645e-05
        DEBUG:root:i=444 residual=1.3743452655035071e-05
        DEBUG:root:i=445 residual=1.3591749848274048e-05
        DEBUG:root:i=446 residual=1.3444052456179634e-05
        DEBUG:root:i=447 residual=1.3298010344442446e-05
        DEBUG:root:i=448 residual=1.3153634426998906e-05
        DEBUG:root:i=449 residual=1.3009213944314979e-05
        DEBUG:root:i=450 residual=1.2866457836935297e-05
        DEBUG:root:i=451 residual=1.2725419765047263e-05
        DEBUG:root:i=452 residual=1.2587205674208235e-05
        DEBUG:root:i=453 residual=1.2453631825337652e-05
        DEBUG:root:i=454 residual=1.231362512044143e-05
        DEBUG:root:i=455 residual=1.2179385521449149e-05
        DEBUG:root:i=456 residual=1.20468548630015e-05
        DEBUG:root:i=457 residual=1.1915713912458159e-05
        DEBUG:root:i=458 residual=1.1784925845859107e-05
        DEBUG:root:i=459 residual=1.1657190952973906e-05
        DEBUG:root:i=460 residual=1.1530019946803804e-05
        DEBUG:root:i=461 residual=1.140332460636273e-05
        DEBUG:root:i=462 residual=1.1279984391876496e-05
        DEBUG:root:i=463 residual=1.1156753316754475e-05
        DEBUG:root:i=464 residual=1.1035427633032668e-05
        DEBUG:root:i=465 residual=1.0914944141404703e-05
        DEBUG:root:i=466 residual=1.079639514500741e-05
        DEBUG:root:i=467 residual=1.067899302142905e-05
        DEBUG:root:i=468 residual=1.0561459930613637e-05
        DEBUG:root:i=469 residual=1.0448122338857502e-05
        DEBUG:root:i=470 residual=1.03326474345522e-05
        DEBUG:root:i=471 residual=1.0220426702289842e-05
        DEBUG:root:i=472 residual=1.0109519280376844e-05
        DEBUG:root:i=473 residual=9.997517736337613e-06
        DEBUG:root:i=474 residual=9.890190085570794e-06
        DEBUG:root:i=475 residual=9.782183951756451e-06
        DEBUG:root:i=476 residual=9.67624055192573e-06
        DEBUG:root:i=477 residual=9.57028260017978e-06
        DEBUG:root:i=478 residual=9.466019946557935e-06
        DEBUG:root:i=479 residual=9.366169251734391e-06
        DEBUG:root:i=480 residual=9.25991298572626e-06
        DEBUG:root:i=481 residual=9.161411981040146e-06
        DEBUG:root:i=482 residual=9.061219316208735e-06
        DEBUG:root:i=483 residual=8.962610991147812e-06
        DEBUG:root:i=484 residual=8.864939445629716e-06
        DEBUG:root:i=485 residual=8.76844842423452e-06
        DEBUG:root:i=486 residual=8.672359399497509e-06
        DEBUG:root:i=487 residual=8.578732376918197e-06
        DEBUG:root:i=488 residual=8.484119462082162e-06
        DEBUG:root:i=489 residual=8.393281859753188e-06
        DEBUG:root:i=490 residual=8.3017703218502e-06
        DEBUG:root:i=491 residual=8.213298315240536e-06
        DEBUG:root:i=492 residual=8.122345207084436e-06
        DEBUG:root:i=493 residual=8.0337331382907e-06
        DEBUG:root:i=494 residual=7.945382094476372e-06
        DEBUG:root:i=495 residual=7.859487595851533e-06
        DEBUG:root:i=496 residual=7.778921826684382e-06
        DEBUG:root:i=497 residual=7.689205631322693e-06
        DEBUG:root:i=498 residual=7.606872259202646e-06
        DEBUG:root:i=499 residual=7.523688509536441e-06
        DEBUG:root:i=500 residual=7.44110684536281e-06
        DEBUG:root:i=501 residual=7.3615087785583455e-06
        DEBUG:root:i=502 residual=7.281331818376202e-06
        DEBUG:root:i=503 residual=7.203375389508437e-06
        DEBUG:root:i=504 residual=7.123986051738029e-06
        DEBUG:root:i=505 residual=7.046387963782763e-06
        DEBUG:root:i=506 residual=6.969655260036234e-06
        DEBUG:root:i=507 residual=6.894273610669188e-06
        DEBUG:root:i=508 residual=6.819269856350729e-06
        DEBUG:root:i=509 residual=6.744757229171228e-06
        DEBUG:root:i=510 residual=6.670717993983999e-06
        DEBUG:root:i=511 residual=6.5988883761747275e-06
        DEBUG:root:i=512 residual=6.529469374072505e-06
        DEBUG:root:i=513 residual=6.458086318161804e-06
        DEBUG:root:i=514 residual=6.386462700902484e-06
        DEBUG:root:i=515 residual=6.318886335066054e-06
        DEBUG:root:i=516 residual=6.249180387385422e-06
        DEBUG:root:i=517 residual=6.1828595789847896e-06
        DEBUG:root:i=518 residual=6.1144451137806755e-06
        DEBUG:root:i=519 residual=6.049062903912272e-06
        DEBUG:root:i=520 residual=5.9809635786223225e-06
        DEBUG:root:i=521 residual=5.917106591368793e-06
        DEBUG:root:i=522 residual=5.8539631027088035e-06
        DEBUG:root:i=523 residual=5.788610906165559e-06
        DEBUG:root:i=524 residual=5.726822564611211e-06
        DEBUG:root:i=525 residual=5.663331194227794e-06
        DEBUG:root:i=526 residual=5.602377314062323e-06
        DEBUG:root:i=527 residual=5.543129191210028e-06
        DEBUG:root:i=528 residual=5.481153493747115e-06
        DEBUG:root:i=529 residual=5.422243248176528e-06
        DEBUG:root:i=530 residual=5.363051968743093e-06
        DEBUG:root:i=531 residual=5.304979367792839e-06
        DEBUG:root:i=532 residual=5.2472532843239605e-06
        DEBUG:root:i=533 residual=5.190038791624829e-06
        DEBUG:root:i=534 residual=5.135286301083397e-06
        DEBUG:root:i=535 residual=5.078846697870176e-06
        DEBUG:root:i=536 residual=5.022950062993914e-06
        DEBUG:root:i=537 residual=4.968299890606431e-06
        DEBUG:root:i=538 residual=4.915227691526525e-06
        DEBUG:root:i=539 residual=4.861195520788897e-06
        DEBUG:root:i=540 residual=4.808141511603026e-06
        DEBUG:root:i=541 residual=4.7565940803906415e-06
        DEBUG:root:i=542 residual=4.706119398178998e-06
        DEBUG:root:i=543 residual=4.654755230149021e-06
        DEBUG:root:i=544 residual=4.603833531291457e-06
        DEBUG:root:i=545 residual=4.554462975647766e-06
        DEBUG:root:i=546 residual=4.504268417804269e-06
        DEBUG:root:i=547 residual=4.455523594515398e-06
        DEBUG:root:i=548 residual=4.4066091504646465e-06
        DEBUG:root:i=549 residual=4.359340437076753e-06
        DEBUG:root:i=550 residual=4.311761585995555e-06
        DEBUG:root:i=551 residual=4.2650972318369895e-06
        DEBUG:root:i=552 residual=4.219451966491761e-06
        DEBUG:root:i=553 residual=4.1730108932824805e-06
        DEBUG:root:i=554 residual=4.132204139750684e-06
        DEBUG:root:i=555 residual=4.088300102011999e-06
        DEBUG:root:i=556 residual=4.041223746753531e-06
        DEBUG:root:i=557 residual=3.996011400886346e-06
        DEBUG:root:i=558 residual=3.953571194870165e-06
        DEBUG:root:i=559 residual=3.91051344195148e-06
        DEBUG:root:i=560 residual=3.867212399200071e-06
        DEBUG:root:i=561 residual=3.826037755061407e-06
        DEBUG:root:i=562 residual=3.784243062909809e-06
        DEBUG:root:i=563 residual=3.7427885217766743e-06
        DEBUG:root:i=564 residual=3.703125003085006e-06
        DEBUG:root:i=565 residual=3.6617802834371105e-06
        DEBUG:root:i=566 residual=3.621431233113981e-06
        DEBUG:root:i=567 residual=3.582340468710754e-06
        DEBUG:root:i=568 residual=3.544490937201772e-06
        DEBUG:root:i=569 residual=3.506020220811479e-06
        DEBUG:root:i=570 residual=3.4678121210163226e-06
        DEBUG:root:i=571 residual=3.4314482491026865e-06
        DEBUG:root:i=572 residual=3.394102350284811e-06
        DEBUG:root:i=573 residual=3.356242586960434e-06
        DEBUG:root:i=574 residual=3.319063580420334e-06
        DEBUG:root:i=575 residual=3.2841014672158053e-06
        DEBUG:root:i=576 residual=3.2497921438334743e-06
        DEBUG:root:i=577 residual=3.213293894077651e-06
        DEBUG:root:i=578 residual=3.1783024496689904e-06
        DEBUG:root:i=579 residual=3.1435986329597654e-06
        DEBUG:root:i=580 residual=3.1102076718525495e-06
        DEBUG:root:i=581 residual=3.084524450969184e-06
        DEBUG:root:i=582 residual=3.042908247152809e-06
        DEBUG:root:i=583 residual=3.0106295980658615e-06
        DEBUG:root:i=584 residual=2.976075847982429e-06
        DEBUG:root:i=585 residual=2.946338327092235e-06
        DEBUG:root:i=586 residual=2.9123448257450946e-06
        DEBUG:root:i=587 residual=2.8825286335631972e-06
        DEBUG:root:i=588 residual=2.8513522920547985e-06
        DEBUG:root:i=589 residual=2.820150939442101e-06
        DEBUG:root:i=590 residual=2.7901116936845938e-06
        DEBUG:root:i=591 residual=2.7595501705945935e-06
        DEBUG:root:i=592 residual=2.728439085331047e-06
        DEBUG:root:i=593 residual=2.700315462789149e-06
        DEBUG:root:i=594 residual=2.6718678327597445e-06
        DEBUG:root:i=595 residual=2.641698074512533e-06
        DEBUG:root:i=596 residual=2.612714752103784e-06
        DEBUG:root:i=597 residual=2.584586354714702e-06
        DEBUG:root:i=598 residual=2.557664174673846e-06
        DEBUG:root:i=599 residual=2.5287010885222116e-06
        DEBUG:root:i=600 residual=2.501485141692683e-06
        DEBUG:root:i=601 residual=2.474821485520806e-06
        DEBUG:root:i=602 residual=2.4470089101669146e-06
        DEBUG:root:i=603 residual=2.4214275526901474e-06
        DEBUG:root:i=604 residual=2.394443981756922e-06
        DEBUG:root:i=605 residual=2.3691122805757914e-06
        DEBUG:root:i=606 residual=2.3438799416908296e-06
        DEBUG:root:i=607 residual=2.3192687876871787e-06
        DEBUG:root:i=608 residual=2.2960302885621786e-06
        DEBUG:root:i=609 residual=2.268468961119652e-06
        DEBUG:root:i=610 residual=2.247153815915226e-06
        DEBUG:root:i=611 residual=2.2197666567080887e-06
        DEBUG:root:i=612 residual=2.1947639652353246e-06
        DEBUG:root:i=613 residual=2.171784217352979e-06
        DEBUG:root:i=614 residual=2.1481660041899886e-06
        DEBUG:root:i=615 residual=2.1238256522337906e-06
        DEBUG:root:i=616 residual=2.1017035578552168e-06
        DEBUG:root:i=617 residual=2.080130343529163e-06
        DEBUG:root:i=618 residual=2.0558293272188166e-06
        DEBUG:root:i=619 residual=2.035118086496368e-06
        DEBUG:root:i=620 residual=2.011685637626215e-06
        DEBUG:root:i=621 residual=1.9920960312447278e-06
        DEBUG:root:i=622 residual=1.968408923858078e-06
        DEBUG:root:i=623 residual=1.9484482436382677e-06
        DEBUG:root:i=624 residual=1.927034645632375e-06
        DEBUG:root:i=625 residual=1.903354245769151e-06
        DEBUG:root:i=626 residual=1.8848928675652132e-06
        DEBUG:root:i=627 residual=1.8648157720235758e-06
        DEBUG:root:i=628 residual=1.8447710772306891e-06
        DEBUG:root:i=629 residual=1.8243389376948471e-06
        DEBUG:root:i=630 residual=1.8042399005935295e-06
        DEBUG:root:i=631 residual=1.7903255411511054e-06
        DEBUG:root:i=632 residual=1.7672318790573627e-06
        DEBUG:root:i=633 residual=1.7500555031801923e-06
        DEBUG:root:i=634 residual=1.7308096857959754e-06
        DEBUG:root:i=635 residual=1.7128412537203985e-06
        DEBUG:root:i=636 residual=1.6909808664422599e-06
        DEBUG:root:i=637 residual=1.6738881640776526e-06
        DEBUG:root:i=638 residual=1.6557966091568233e-06
        DEBUG:root:i=639 residual=1.6362808992198552e-06
        DEBUG:root:i=640 residual=1.6179930071302806e-06
        DEBUG:root:i=641 residual=1.6014919310691766e-06
        DEBUG:root:i=642 residual=1.5859955055930186e-06
        DEBUG:root:i=643 residual=1.5672285371692851e-06
        DEBUG:root:i=644 residual=1.550790784676792e-06
        DEBUG:root:i=645 residual=1.5377654563053511e-06
        DEBUG:root:i=646 residual=1.5260876580214244e-06
        DEBUG:root:i=647 residual=1.503594603491365e-06
        DEBUG:root:i=648 residual=1.4831118733127369e-06
        DEBUG:root:i=649 residual=1.468720483899233e-06
        DEBUG:root:i=650 residual=1.4564999446520233e-06
        DEBUG:root:i=651 residual=1.4379874073711107e-06
        DEBUG:root:i=652 residual=1.4211528878149693e-06
        DEBUG:root:i=653 residual=1.4076056231715484e-06
        DEBUG:root:i=654 residual=1.390565671499644e-06
        DEBUG:root:i=655 residual=1.3766396023129346e-06
        DEBUG:root:i=656 residual=1.362696025353216e-06
        DEBUG:root:i=657 residual=1.3466877817336353e-06
        DEBUG:root:i=658 residual=1.3309678479345166e-06
        DEBUG:root:i=659 residual=1.3159411764718243e-06
        DEBUG:root:i=660 residual=1.3018988056501257e-06
        DEBUG:root:i=661 residual=1.2877704875791096e-06
        DEBUG:root:i=662 residual=1.2746759239234962e-06
        DEBUG:root:i=663 residual=1.2621732139450614e-06
        DEBUG:root:i=664 residual=1.2526697901193984e-06
        DEBUG:root:i=665 residual=1.2328425782470731e-06
        DEBUG:root:i=666 residual=1.223842332365166e-06
        DEBUG:root:i=667 residual=1.2059093705829582e-06
        DEBUG:root:i=668 residual=1.1950755833822768e-06
        DEBUG:root:i=669 residual=1.183338667942735e-06
        DEBUG:root:i=670 residual=1.1703110658345395e-06
        DEBUG:root:i=671 residual=1.1573677056730958e-06
        DEBUG:root:i=672 residual=1.1454495734142256e-06
        DEBUG:root:i=673 residual=1.131956082645047e-06
        DEBUG:root:i=674 residual=1.1185400126123568e-06
        DEBUG:root:i=675 residual=1.107506250264123e-06
        DEBUG:root:i=676 residual=1.098294433177216e-06
        DEBUG:root:i=677 residual=1.0824064702319447e-06
        DEBUG:root:i=678 residual=1.0730774420153466e-06
        DEBUG:root:i=679 residual=1.0612884580041282e-06
        DEBUG:root:i=680 residual=1.050175683303678e-06
        DEBUG:root:i=681 residual=1.03606441825832e-06
        DEBUG:root:i=682 residual=1.0244245913781924e-06
        DEBUG:root:i=683 residual=1.014870690596581e-06
        DEBUG:root:i=684 residual=1.0036894764198223e-06
        DEBUG:root:i=685 residual=9.915290775097674e-07
        INFO:root:rank = 0 ranking = 7.0149e-01 url= www.lawfareblog.com/covid-19-speech-and-surveillance-response
        INFO:root:rank = 1 ranking = 7.0149e-01 url= www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
        INFO:root:rank = 2 ranking = 1.0552e-01 url= www.lawfareblog.com/cost-using-zero-days
        INFO:root:rank = 3 ranking = 3.1757e-02 url= www.lawfareblog.com/lawfare-podcast-former-congressman-brian-baird-and-daniel-schuman-how-congress-can-continue-function
        INFO:root:rank = 4 ranking = 2.2040e-02 url= www.lawfareblog.com/events
        INFO:root:rank = 5 ranking = 1.6027e-02 url= www.lawfareblog.com/water-wars-increased-us-focus-indo-pacific
        INFO:root:rank = 6 ranking = 1.6026e-02 url= www.lawfareblog.com/water-wars-drill-maybe-drill
        INFO:root:rank = 7 ranking = 1.6023e-02 url= www.lawfareblog.com/water-wars-disjointed-operations-south-china-sea
        INFO:root:rank = 8 ranking = 1.6020e-02 url= www.lawfareblog.com/water-wars-song-oil-and-fire
        INFO:root:rank = 9 ranking = 1.6020e-02 url= www.lawfareblog.com/water-wars-sinking-feeling-philippine-china-relations

   ```

   Task 2, part 1:
   ```
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona'
        n= 16862
        v= tensor([0., 0., 0.,  ..., 0., 0., 0.])
        INFO:root:rank = 0 ranking = 6.3217e-01 url= www.lawfareblog.com/covid-19-speech-and-surveillance-response
        INFO:root:rank = 1 ranking = 6.3215e-01 url= www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
        INFO:root:rank = 2 ranking = 1.5565e-01 url= www.lawfareblog.com/chinatalk-how-party-takes-its-propaganda-global
        INFO:root:rank = 3 ranking = 1.1977e-01 url= www.lawfareblog.com/rational-security-my-corona-edition
        INFO:root:rank = 4 ranking = 1.1977e-01 url= www.lawfareblog.com/brexit-not-immune-coronavirus
        INFO:root:rank = 5 ranking = 9.1832e-02 url= www.lawfareblog.com/trump-cant-reopen-country-over-state-objections
        INFO:root:rank = 6 ranking = 8.9400e-02 url= www.lawfareblog.com/britains-coronavirus-response
        INFO:root:rank = 7 ranking = 8.9400e-02 url= www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
        INFO:root:rank = 8 ranking = 7.5428e-02 url= www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
        INFO:root:rank = 9 ranking = 7.1328e-02 url= www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response

   ```

   Task 2, part 2:
   ```
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona' --search_query='-corona'
        n= 16862
        v= tensor([0., 0., 0.,  ..., 0., 0., 0.])
        INFO:root:rank = 0 ranking = 0.0000e+00 url= www.lawfareblog.com/covid-19-speech-and-surveillance-response
        INFO:root:rank = 1 ranking = 0.0000e+00 url= www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
        INFO:root:rank = 2 ranking = 0.0000e+00 url= www.lawfareblog.com/chinatalk-how-party-takes-its-propaganda-global
        INFO:root:rank = 3 ranking = 0.0000e+00 url= www.lawfareblog.com/trump-cant-reopen-country-over-state-objections
        INFO:root:rank = 4 ranking = 0.0000e+00 url= www.lawfareblog.com/lawfare-podcast-mom-and-dad-talk-clinical-trials-pandemic
        INFO:root:rank = 5 ranking = 0.0000e+00 url= www.lawfareblog.com/fault-lines-foreign-policy-quarantined
        INFO:root:rank = 6 ranking = 0.0000e+00 url= www.lawfareblog.com/limits-world-health-organization
        INFO:root:rank = 7 ranking = 0.0000e+00 url= www.lawfareblog.com/chinatalk-dispatches-shanghai-beijing-and-hong-kong
        INFO:root:rank = 8 ranking = 0.0000e+00 url= www.lawfareblog.com/us-moves-dismiss-case-against-company-linked-ira-troll-farm
        INFO:root:rank = 9 ranking = 0.0000e+00 url= www.lawfareblog.com/livestream-house-armed-services-holds-hearing-national-security-challenges-north-and-south-america
   ```

1. Ensure that all your changes to the `pagerank.py` and `README.md` files are committed to your repo and pushed to github.

1. Get at least 5 stars on your repo.
   (You may trade stars with other students in the class.)

   > **NOTE:**
   > 
   > Recruiters use github profiles to determine who to hire,
   > and pagerank is used to rank user profiles and projects.
   > Links in this graph correspond to who has starred/followed who's repo.
   > By getting more stars on your repo, you'll be increasing your github pagerank, which increases the likelihood that recruiters will hire you.
   > To see an example, [perform a search for `data mining`](https://github.com/search?q=data+mining).
   > Notice that the results are returned "approximately" ranked by the number of stars,
   > but because "some stars count more than others" the results are not exactly ranked by the number of stars.
   > (I asked you not to fork this repo because forks are ranked lower than non-forks.)
   >
   > In some sense, we are doing a "dual problem" to data mining by getting these stars.
   > Recruiters are using data mining to find out who the best people to recruit are,
   > and we are hacking their data mining algorithms by making those algorithms select you instead of someone else.
   >
   > If you're interested in exploring this idea further, here's a python tutorial for extracting GitHub's social graph: <https://www.oreilly.com/library/view/mining-the-social/9781449368180/ch07.html> ; if you're interested in learning more about how recruiters use github profiles, read this Hacker News post: <https://news.ycombinator.com/item?id=19413348>.

1. Submit the url of your repo to sakai.

   Each part is worth 2 points, for 12 points overall.
