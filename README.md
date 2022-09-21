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
        DEBUG:root:i=0 residual=0.3775096535682678
        DEBUG:root:i=1 residual=0.3134882152080536
        DEBUG:root:i=2 residual=0.2756592035293579
        DEBUG:root:i=3 residual=0.21698090434074402
        DEBUG:root:i=4 residual=0.18984204530715942
        DEBUG:root:i=5 residual=0.15531454980373383
        DEBUG:root:i=6 residual=0.13266263902187347
        DEBUG:root:i=7 residual=0.11062280833721161
        DEBUG:root:i=8 residual=0.0935136154294014
        DEBUG:root:i=9 residual=0.07847193628549576
        DEBUG:root:i=10 residual=0.06611069291830063
        DEBUG:root:i=11 residual=0.05558090656995773
        DEBUG:root:i=12 residual=0.04677915573120117
        DEBUG:root:i=13 residual=0.039349012076854706
        DEBUG:root:i=14 residual=0.03310869261622429
        DEBUG:root:i=15 residual=0.027853948995471
        DEBUG:root:i=16 residual=0.023434894159436226
        DEBUG:root:i=17 residual=0.019716130569577217
        DEBUG:root:i=18 residual=0.016587838530540466
        DEBUG:root:i=19 residual=0.013955758884549141
        DEBUG:root:i=20 residual=0.011741633526980877
        DEBUG:root:i=21 residual=0.009878222830593586
        DEBUG:root:i=22 residual=0.008310933597385883
        DEBUG:root:i=23 residual=0.006992380600422621
        DEBUG:root:i=24 residual=0.00588278379291296
        DEBUG:root:i=25 residual=0.00494928564876318
        DEBUG:root:i=26 residual=0.004164101090282202
        DEBUG:root:i=27 residual=0.003503275103867054
        DEBUG:root:i=28 residual=0.002947412896901369
        DEBUG:root:i=29 residual=0.0024798011872917414
        DEBUG:root:i=30 residual=0.002086422871798277
        DEBUG:root:i=31 residual=0.001755191246047616
        DEBUG:root:i=32 residual=0.0014767624670639634
        DEBUG:root:i=33 residual=0.0012423915322870016
        DEBUG:root:i=34 residual=0.0010452595306560397
        DEBUG:root:i=35 residual=0.0008794325985945761
        DEBUG:root:i=36 residual=0.0007398635498248041
        DEBUG:root:i=37 residual=0.0006225021206773818
        DEBUG:root:i=38 residual=0.0005237152217887342
        DEBUG:root:i=39 residual=0.00044049162534065545
        DEBUG:root:i=40 residual=0.0003707956930156797
        DEBUG:root:i=41 residual=0.0003118595341220498
        DEBUG:root:i=42 residual=0.00026235461700707674
        DEBUG:root:i=43 residual=0.0002207769575761631
        DEBUG:root:i=44 residual=0.00018579662719275802
        DEBUG:root:i=45 residual=0.0001563065015943721
        DEBUG:root:i=46 residual=0.0001315098925260827
        DEBUG:root:i=47 residual=0.00011065506259910762
        DEBUG:root:i=48 residual=9.283208783017471e-05
        DEBUG:root:i=49 residual=7.835238648112863e-05
        DEBUG:root:i=50 residual=6.579793989658356e-05
        DEBUG:root:i=51 residual=5.543866427615285e-05
        DEBUG:root:i=52 residual=4.6715060307178646e-05
        DEBUG:root:i=53 residual=3.909929364454001e-05
        DEBUG:root:i=54 residual=3.294386260677129e-05
        DEBUG:root:i=55 residual=2.7852673156303354e-05
        DEBUG:root:i=56 residual=2.3248065190273337e-05
        DEBUG:root:i=57 residual=1.966203490155749e-05
        DEBUG:root:i=58 residual=1.6471843991894275e-05
        DEBUG:root:i=59 residual=1.4014856787980534e-05
        DEBUG:root:i=60 residual=1.1800590982602444e-05
        DEBUG:root:i=61 residual=9.742259862832725e-06
        DEBUG:root:i=62 residual=8.302875357912853e-06
        DEBUG:root:i=63 residual=7.063716111588292e-06
        DEBUG:root:i=64 residual=5.845966825290816e-06
        DEBUG:root:i=65 residual=4.962869752489496e-06
        DEBUG:root:i=66 residual=4.206880475976504e-06
        DEBUG:root:i=67 residual=3.499711510812631e-06
        DEBUG:root:i=68 residual=2.992129338963423e-06
        DEBUG:root:i=69 residual=2.5033950805664062e-06
        DEBUG:root:i=70 residual=2.214214191553765e-06
        DEBUG:root:i=71 residual=1.955177822310361e-06
        DEBUG:root:i=72 residual=1.3902072169003077e-06
        DEBUG:root:i=73 residual=1.244581540049694e-06
        DEBUG:root:i=74 residual=9.97376446321141e-07
        INFO:root:rank=0 pagerank=2.1634e+00 url=4
        INFO:root:rank=1 pagerank=1.6664e+00 url=6
        INFO:root:rank=2 pagerank=1.2402e+00 url=5
        INFO:root:rank=3 pagerank=4.5712e-01 url=2
        INFO:root:rank=4 pagerank=3.5620e-01 url=3
        INFO:root:rank=5 pagerank=3.2078e-01 url=1

   ```

   Task 1, part 2:
   ```
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='corona'
        INFO:root:rank=0 pagerank=4.5861e-03 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
        INFO:root:rank=1 pagerank=4.0460e-03 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
        INFO:root:rank=2 pagerank=2.6116e-03 url=www.lawfareblog.com/britains-coronavirus-response
        INFO:root:rank=3 pagerank=2.5390e-03 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
        INFO:root:rank=4 pagerank=2.3557e-03 url=www.lawfareblog.com/israeli-emergency-regulations-location-tracking-coronavirus-carriers
        INFO:root:rank=5 pagerank=2.2895e-03 url=www.lawfareblog.com/why-congress-conducting-business-usual-face-coronavirus
        INFO:root:rank=6 pagerank=2.2727e-03 url=www.lawfareblog.com/livestream-house-oversight-committee-holds-hearing-government-coronavirus-response
        INFO:root:rank=7 pagerank=2.2520e-03 url=www.lawfareblog.com/congressional-homeland-security-committees-seek-ways-support-state-federal-responses-coronavirus
        INFO:root:rank=8 pagerank=2.1878e-03 url=www.lawfareblog.com/paper-hearing-experts-debate-digital-contact-tracing-and-coronavirus-privacy-concerns
        INFO:root:rank=9 pagerank=2.0339e-03 url=www.lawfareblog.com/cyberlaw-podcast-how-israel-fighting-coronavirus

   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='trump'
        INFO:root:rank=0 pagerank=6.6243e-02 url=www.lawfareblog.com/donald-trump-and-politically-weaponized-executive-branch
        INFO:root:rank=1 pagerank=6.0194e-02 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
        INFO:root:rank=2 pagerank=3.4969e-02 url=www.lawfareblog.com/trump-administrations-worrying-new-policy-israeli-settlements
        INFO:root:rank=3 pagerank=3.2193e-02 url=www.lawfareblog.com/document-trump-revokes-obama-executive-order-counterterrorism-strike-casualty-reporting
        INFO:root:rank=4 pagerank=3.0971e-02 url=www.lawfareblog.com/dc-circuit-overrules-district-courts-due-process-ruling-qasim-v-trump
        INFO:root:rank=5 pagerank=2.8460e-02 url=www.lawfareblog.com/how-trumps-approach-middle-east-ignores-past-future-and-human-condition
        INFO:root:rank=6 pagerank=2.5252e-02 url=www.lawfareblog.com/why-trump-cant-buy-greenland
        INFO:root:rank=7 pagerank=2.2457e-02 url=www.lawfareblog.com/oral-argument-summary-qassim-v-trump
        INFO:root:rank=8 pagerank=2.1462e-02 url=www.lawfareblog.com/dc-circuit-court-denies-trump-rehearing-mazars-case
        INFO:root:rank=9 pagerank=2.1103e-02 url=www.lawfareblog.com/second-circuit-rules-mazars-must-hand-over-trump-tax-returns-new-york-prosecutors

   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='iran'
        INFO:root:rank=0 pagerank=6.6131e-02 url=www.lawfareblog.com/praise-presidents-iran-tweets
        INFO:root:rank=1 pagerank=2.9199e-02 url=www.lawfareblog.com/how-us-iran-tensions-could-disrupt-iraqs-fragile-peace
        INFO:root:rank=2 pagerank=1.7709e-02 url=www.lawfareblog.com/cyber-command-operational-update-clarifying-june-2019-iran-operation
        INFO:root:rank=3 pagerank=1.4604e-02 url=www.lawfareblog.com/aborted-iran-strike-fine-line-between-necessity-and-revenge
        INFO:root:rank=4 pagerank=8.4512e-03 url=www.lawfareblog.com/iranian-hostage-crisis-and-its-effect-american-politics
        INFO:root:rank=5 pagerank=8.3989e-03 url=www.lawfareblog.com/parsing-state-departments-letter-use-force-against-iran
        INFO:root:rank=6 pagerank=8.2581e-03 url=www.lawfareblog.com/announcing-united-states-and-use-force-against-iran-new-lawfare-e-book
        INFO:root:rank=7 pagerank=8.0561e-03 url=www.lawfareblog.com/trump-moves-cut-irans-oil-revenues-whats-his-endgame
        INFO:root:rank=8 pagerank=7.1939e-03 url=www.lawfareblog.com/us-names-iranian-revolutionary-guard-terrorist-organization-and-sanctions-international-criminal
        INFO:root:rank=9 pagerank=5.9405e-03 url=www.lawfareblog.com/iran-shoots-down-us-drone-domestic-and-international-legal-implications
   ```

   Task 1, part 3:
   ```
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz
        INFO:root:rank=0 pagerank=8.4156e+00 url=www.lawfareblog.com/lawfare-job-board
        INFO:root:rank=1 pagerank=8.4156e+00 url=www.lawfareblog.com/masthead
        INFO:root:rank=2 pagerank=8.4156e+00 url=www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
        INFO:root:rank=3 pagerank=8.4156e+00 url=www.lawfareblog.com/documents-related-mueller-investigation
        INFO:root:rank=4 pagerank=8.4156e+00 url=www.lawfareblog.com/topics
        INFO:root:rank=5 pagerank=8.4156e+00 url=www.lawfareblog.com/about-lawfare-brief-history-term-and-site
        INFO:root:rank=6 pagerank=8.4156e+00 url=www.lawfareblog.com/snowden-revelations
        INFO:root:rank=7 pagerank=8.4156e+00 url=www.lawfareblog.com/support-lawfare
        INFO:root:rank=8 pagerank=8.4156e+00 url=www.lawfareblog.com/upcoming-events
        INFO:root:rank=9 pagerank=8.4156e+00 url=www.lawfareblog.com/our-comments-policy

   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2
        INFO:root:rank=0 pagerank=4.6091e+00 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
        INFO:root:rank=1 pagerank=2.9867e+00 url=www.lawfareblog.com/livestream-nov-21-impeachment-hearings-0
        INFO:root:rank=2 pagerank=2.9669e+00 url=www.lawfareblog.com/opening-statement-david-holmes
        INFO:root:rank=3 pagerank=2.0173e+00 url=www.lawfareblog.com/senate-examines-threats-homeland
        INFO:root:rank=4 pagerank=1.8769e+00 url=www.lawfareblog.com/what-make-first-day-impeachment-hearings
        INFO:root:rank=5 pagerank=1.8762e+00 url=www.lawfareblog.com/livestream-house-armed-services-committee-hearing-f-35-program
        INFO:root:rank=6 pagerank=1.8693e+00 url=www.lawfareblog.com/whats-house-resolution-impeachment
        INFO:root:rank=7 pagerank=1.7655e+00 url=www.lawfareblog.com/congress-us-policy-toward-syria-and-turkey-overview-recent-hearings
        INFO:root:rank=8 pagerank=1.6807e+00 url=www.lawfareblog.com/summary-david-holmess-deposition-testimony
        INFO:root:rank=9 pagerank=9.8346e-01 url=www.lawfareblog.com/events
   ```

   Task 1, part 4:
   ```
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose 
        DEBUG:root:computing indices
        DEBUG:root:computing values
        DEBUG:root:i=0 residual=20.519933700561523
        DEBUG:root:i=1 residual=6.110218524932861
        DEBUG:root:i=2 residual=1.9214706420898438
        DEBUG:root:i=3 residual=0.5871003866195679
        DEBUG:root:i=4 residual=0.17524230480194092
        DEBUG:root:i=5 residual=0.05138478800654411
        DEBUG:root:i=6 residual=0.014827270992100239
        DEBUG:root:i=7 residual=0.00416865898296237
        DEBUG:root:i=8 residual=0.001077273627743125
        DEBUG:root:i=9 residual=0.0001930922007886693
        DEBUG:root:i=10 residual=8.493969653500244e-05
        DEBUG:root:i=11 residual=0.00013263747678138316
        DEBUG:root:i=12 residual=0.000128910323837772
        DEBUG:root:i=13 residual=0.00010905414092121646
        DEBUG:root:i=14 residual=9.583001519786194e-05
        DEBUG:root:i=15 residual=7.931196159915999e-05
        DEBUG:root:i=16 residual=6.609127012779936e-05
        DEBUG:root:i=17 residual=5.617739225272089e-05
        DEBUG:root:i=18 residual=4.626503505278379e-05
        DEBUG:root:i=19 residual=3.96539326175116e-05
        DEBUG:root:i=20 residual=3.634970198618248e-05
        DEBUG:root:i=21 residual=3.3044216252164915e-05
        DEBUG:root:i=22 residual=2.6437381166033447e-05
        DEBUG:root:i=23 residual=1.982917638088111e-05
        DEBUG:root:i=24 residual=1.6522233636351302e-05
        DEBUG:root:i=25 residual=1.652348873903975e-05
        DEBUG:root:i=26 residual=1.3217977539170533e-05
        DEBUG:root:i=27 residual=9.91389879345661e-06
        DEBUG:root:i=28 residual=9.912546374835074e-06
        DEBUG:root:i=29 residual=9.912531822919846e-06
        DEBUG:root:i=30 residual=3.3123531011369778e-06
        DEBUG:root:i=31 residual=6.608392141060904e-06
        DEBUG:root:i=32 residual=6.608376224903623e-06
        DEBUG:root:i=33 residual=3.3059432098525576e-06
        DEBUG:root:i=34 residual=3.3041985716408817e-06
        DEBUG:root:i=35 residual=6.144799158391834e-08
        INFO:root:rank=0 pagerank=8.4156e+00 url=www.lawfareblog.com/lawfare-job-board
        INFO:root:rank=1 pagerank=8.4156e+00 url=www.lawfareblog.com/masthead
        INFO:root:rank=2 pagerank=8.4156e+00 url=www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
        INFO:root:rank=3 pagerank=8.4156e+00 url=www.lawfareblog.com/documents-related-mueller-investigation
        INFO:root:rank=4 pagerank=8.4156e+00 url=www.lawfareblog.com/topics
        INFO:root:rank=5 pagerank=8.4156e+00 url=www.lawfareblog.com/about-lawfare-brief-history-term-and-site
        INFO:root:rank=6 pagerank=8.4156e+00 url=www.lawfareblog.com/snowden-revelations
        INFO:root:rank=7 pagerank=8.4156e+00 url=www.lawfareblog.com/support-lawfare
        INFO:root:rank=8 pagerank=8.4156e+00 url=www.lawfareblog.com/upcoming-events
        INFO:root:rank=9 pagerank=8.4156e+00 url=www.lawfareblog.com/our-comments-policy
   
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --alpha=0.99999
        DEBUG:root:computing indices
        DEBUG:root:computing values
        DEBUG:root:i=0 residual=24.14374542236328
        DEBUG:root:i=1 residual=8.458536148071289
        DEBUG:root:i=2 residual=3.128673791885376
        DEBUG:root:i=3 residual=1.1249767541885376
        DEBUG:root:i=4 residual=0.3952814042568207
        DEBUG:root:i=5 residual=0.1368953287601471
        DEBUG:root:i=6 residual=0.04696769639849663
        DEBUG:root:i=7 residual=0.01595989428460598
        DEBUG:root:i=8 residual=0.005269113462418318
        DEBUG:root:i=9 residual=0.0015950000379234552
        DEBUG:root:i=10 residual=0.00036291309515945613
        DEBUG:root:i=11 residual=0.00015495572006329894
        DEBUG:root:i=12 residual=0.0002592563396319747
        DEBUG:root:i=13 residual=0.0002975475799757987
        DEBUG:root:i=14 residual=0.00030072996742092073
        DEBUG:root:i=15 residual=0.000310623727273196
        DEBUG:root:i=16 residual=0.0003238389326725155
        DEBUG:root:i=17 residual=0.0003172406868543476
        DEBUG:root:i=18 residual=0.00031723949359729886
        DEBUG:root:i=19 residual=0.00031723809661343694
        DEBUG:root:i=20 residual=0.00031723809661343694
        DEBUG:root:i=21 residual=0.0003172396100126207
        DEBUG:root:i=22 residual=0.000317236699629575
        DEBUG:root:i=23 residual=0.00031723809661343694
        DEBUG:root:i=24 residual=0.0003172380675096065
        DEBUG:root:i=25 residual=0.000317238038405776
        DEBUG:root:i=26 residual=0.00031723809661343694
        DEBUG:root:i=27 residual=0.0003172394644934684
        DEBUG:root:i=28 residual=0.0003172381839249283
        DEBUG:root:i=29 residual=0.00031723809661343694
        DEBUG:root:i=30 residual=0.0003172381257172674
        DEBUG:root:i=31 residual=0.0003172380675096065
        DEBUG:root:i=32 residual=0.0003172381257172674
        DEBUG:root:i=33 residual=0.0003172380675096065
        DEBUG:root:i=34 residual=0.0003172395518049598
        DEBUG:root:i=35 residual=0.00031723809661343694
        DEBUG:root:i=36 residual=0.00031723809661343694
        DEBUG:root:i=37 residual=0.00031723809661343694
        DEBUG:root:i=38 residual=0.0003172380675096065
        DEBUG:root:i=39 residual=0.0003172382421325892
        DEBUG:root:i=40 residual=0.000317238038405776
        DEBUG:root:i=41 residual=0.00031723958090879023
        DEBUG:root:i=42 residual=0.00031723672873340547
        DEBUG:root:i=43 residual=0.00031723809661343694
        DEBUG:root:i=44 residual=0.00031723809661343694
        DEBUG:root:i=45 residual=0.0003139353939332068
        DEBUG:root:i=46 residual=0.00031723672873340547
        DEBUG:root:i=47 residual=0.00031723795109428465
        DEBUG:root:i=48 residual=0.00032054222538135946
        DEBUG:root:i=49 residual=0.00032054216717369854
        DEBUG:root:i=50 residual=0.00031723958090879023
        DEBUG:root:i=51 residual=0.00031723809661343694
        DEBUG:root:i=52 residual=0.00031723963911645114
        DEBUG:root:i=53 residual=0.00031723809661343694
        DEBUG:root:i=54 residual=0.000317238038405776
        DEBUG:root:i=55 residual=0.0003172381839249283
        DEBUG:root:i=56 residual=0.0003139353939332068
        DEBUG:root:i=57 residual=0.0003139341133646667
        DEBUG:root:i=58 residual=0.0003172365832142532
        DEBUG:root:i=59 residual=0.00031723809661343694
        DEBUG:root:i=60 residual=0.00031723672873340547
        DEBUG:root:i=61 residual=0.00031723949359729886
        DEBUG:root:i=62 residual=0.00031723809661343694
        DEBUG:root:i=63 residual=0.00031723809661343694
        DEBUG:root:i=64 residual=0.000317238038405776
        DEBUG:root:i=65 residual=0.00031723809661343694
        DEBUG:root:i=66 residual=0.0003172380675096065
        DEBUG:root:i=67 residual=0.00031723821302875876
        DEBUG:root:i=68 residual=0.0003172396100126207
        DEBUG:root:i=69 residual=0.0003172380675096065
        DEBUG:root:i=70 residual=0.00031723809661343694
        DEBUG:root:i=71 residual=0.0003172380675096065
        DEBUG:root:i=72 residual=0.0003139354521408677
        DEBUG:root:i=73 residual=0.00031723667052574456
        DEBUG:root:i=74 residual=0.0003172380675096065
        DEBUG:root:i=75 residual=0.000317238038405776
        DEBUG:root:i=76 residual=0.00031723949359729886
        DEBUG:root:i=77 residual=0.0003172381839249283
        DEBUG:root:i=78 residual=0.00031723809661343694
        DEBUG:root:i=79 residual=0.00032053954782895744
        DEBUG:root:i=80 residual=0.00031723943538963795
        DEBUG:root:i=81 residual=0.00031723949359729886
        DEBUG:root:i=82 residual=0.0003205407701898366
        DEBUG:root:i=83 residual=0.0003172395227011293
        DEBUG:root:i=84 residual=0.00031723815482109785
        DEBUG:root:i=85 residual=0.0003172381257172674
        DEBUG:root:i=86 residual=0.0003172394644934684
        DEBUG:root:i=87 residual=0.0003172380675096065
        DEBUG:root:i=88 residual=0.0003172381839249283
        DEBUG:root:i=89 residual=0.0003172380675096065
        DEBUG:root:i=90 residual=0.00031723809661343694
        DEBUG:root:i=91 residual=0.0003172381839249283
        DEBUG:root:i=92 residual=0.0003139353939332068
        DEBUG:root:i=93 residual=0.00031723672873340547
        DEBUG:root:i=94 residual=0.00031723943538963795
        DEBUG:root:i=95 residual=0.0003172381839249283
        DEBUG:root:i=96 residual=0.0003172381257172674
        DEBUG:root:i=97 residual=0.00031393536482937634
        DEBUG:root:i=98 residual=0.0003172366414219141
        DEBUG:root:i=99 residual=0.00031723672873340547
        DEBUG:root:i=100 residual=0.0003172380675096065
        DEBUG:root:i=101 residual=0.0003172381839249283
        DEBUG:root:i=102 residual=0.0003172395518049598
        DEBUG:root:i=103 residual=0.00032054074108600616
        DEBUG:root:i=104 residual=0.00031723949359729886
        DEBUG:root:i=105 residual=0.0003172381839249283
        DEBUG:root:i=106 residual=0.0003205407701898366
        DEBUG:root:i=107 residual=0.0003172408905811608
        DEBUG:root:i=108 residual=0.00031723815482109785
        DEBUG:root:i=109 residual=0.0003172380675096065
        DEBUG:root:i=110 residual=0.00031723815482109785
        DEBUG:root:i=111 residual=0.00031723809661343694
        DEBUG:root:i=112 residual=0.0003172395518049598
        DEBUG:root:i=113 residual=0.00031723815482109785
        DEBUG:root:i=114 residual=0.0003172379801981151
        DEBUG:root:i=115 residual=0.0003172381839249283
        DEBUG:root:i=116 residual=0.0003172381257172674
        DEBUG:root:i=117 residual=0.0003172382421325892
        DEBUG:root:i=118 residual=0.0003172381257172674
        DEBUG:root:i=119 residual=0.00031723949359729886
        DEBUG:root:i=120 residual=0.000317238038405776
        DEBUG:root:i=121 residual=0.00031723809661343694
        DEBUG:root:i=122 residual=0.00031723672873340547
        DEBUG:root:i=123 residual=0.00031723809661343694
        DEBUG:root:i=124 residual=0.00031723809661343694
        DEBUG:root:i=125 residual=0.00031723943538963795
        DEBUG:root:i=126 residual=0.0003172381257172674
        DEBUG:root:i=127 residual=0.0003172380675096065
        DEBUG:root:i=128 residual=0.0003172381257172674
        DEBUG:root:i=129 residual=0.00031723809661343694
        DEBUG:root:i=130 residual=0.00031723809661343694
        DEBUG:root:i=131 residual=0.0003172380675096065
        DEBUG:root:i=132 residual=0.0003172395227011293
        DEBUG:root:i=133 residual=0.0003172381839249283
        DEBUG:root:i=134 residual=0.00031723809661343694
        DEBUG:root:i=135 residual=0.00031723821302875876
        DEBUG:root:i=136 residual=0.00031723809661343694
        DEBUG:root:i=137 residual=0.000317238038405776
        DEBUG:root:i=138 residual=0.00031723949359729886
        DEBUG:root:i=139 residual=0.0003172381839249283
        DEBUG:root:i=140 residual=0.00031723821302875876
        DEBUG:root:i=141 residual=0.0003172380675096065
        DEBUG:root:i=142 residual=0.0003139339678455144
        DEBUG:root:i=143 residual=0.00032053934410214424
        DEBUG:root:i=144 residual=0.0003172395227011293
        DEBUG:root:i=145 residual=0.00031723949359729886
        DEBUG:root:i=146 residual=0.00031723815482109785
        DEBUG:root:i=147 residual=0.0003172381257172674
        DEBUG:root:i=148 residual=0.000317238038405776
        DEBUG:root:i=149 residual=0.0003172381257172674
        DEBUG:root:i=150 residual=0.0003139353939332068
        DEBUG:root:i=151 residual=0.0003238421631976962
        DEBUG:root:i=152 residual=0.00032054499024525285
        DEBUG:root:i=153 residual=0.00031723949359729886
        DEBUG:root:i=154 residual=0.00031723809661343694
        DEBUG:root:i=155 residual=0.00031723963911645114
        DEBUG:root:i=156 residual=0.0003172382421325892
        DEBUG:root:i=157 residual=0.000317238038405776
        DEBUG:root:i=158 residual=0.0003172381839249283
        DEBUG:root:i=159 residual=0.000317238038405776
        DEBUG:root:i=160 residual=0.0003172380675096065
        DEBUG:root:i=161 residual=0.0003139353939332068
        DEBUG:root:i=162 residual=0.00031723672873340547
        DEBUG:root:i=163 residual=0.00031723809661343694
        DEBUG:root:i=164 residual=0.0003172381257172674
        DEBUG:root:i=165 residual=0.0003172381839249283
        DEBUG:root:i=166 residual=0.000317238038405776
        DEBUG:root:i=167 residual=0.00031723809661343694
        DEBUG:root:i=168 residual=0.00031723809661343694
        DEBUG:root:i=169 residual=0.0003172381257172674
        DEBUG:root:i=170 residual=0.00031723943538963795
        DEBUG:root:i=171 residual=0.0003172381839249283
        DEBUG:root:i=172 residual=0.00031723809661343694
        DEBUG:root:i=173 residual=0.0003172380675096065
        DEBUG:root:i=174 residual=0.0003172382421325892
        DEBUG:root:i=175 residual=0.000317238038405776
        DEBUG:root:i=176 residual=0.00031723949359729886
        DEBUG:root:i=177 residual=0.00031723809661343694
        DEBUG:root:i=178 residual=0.0003172381257172674
        DEBUG:root:i=179 residual=0.00031723809661343694
        DEBUG:root:i=180 residual=0.00031723809661343694
        DEBUG:root:i=181 residual=0.000317238038405776
        DEBUG:root:i=182 residual=0.0003205407992936671
        DEBUG:root:i=183 residual=0.0003172394644934684
        DEBUG:root:i=184 residual=0.0003172381257172674
        DEBUG:root:i=185 residual=0.0003172381839249283
        DEBUG:root:i=186 residual=0.00031723809661343694
        DEBUG:root:i=187 residual=0.0003172380675096065
        DEBUG:root:i=188 residual=0.0003172394644934684
        DEBUG:root:i=189 residual=0.00031723821302875876
        DEBUG:root:i=190 residual=0.0003139353939332068
        DEBUG:root:i=191 residual=0.00031723672873340547
        DEBUG:root:i=192 residual=0.00031723809661343694
        DEBUG:root:i=193 residual=0.0003139353939332068
        DEBUG:root:i=194 residual=0.00031723672873340547
        DEBUG:root:i=195 residual=0.00031723809661343694
        DEBUG:root:i=196 residual=0.00031723809661343694
        DEBUG:root:i=197 residual=0.0003139367909170687
        DEBUG:root:i=198 residual=0.0003205380344297737
        DEBUG:root:i=199 residual=0.0003172408032696694
        DEBUG:root:i=200 residual=0.0003172382421325892
        DEBUG:root:i=201 residual=0.00031723809661343694
        DEBUG:root:i=202 residual=0.000317236699629575
        DEBUG:root:i=203 residual=0.0003172380675096065
        DEBUG:root:i=204 residual=0.00031723949359729886
        DEBUG:root:i=205 residual=0.0003172381839249283
        DEBUG:root:i=206 residual=0.000317238038405776
        DEBUG:root:i=207 residual=0.0003172381257172674
        DEBUG:root:i=208 residual=0.00031723809661343694
        DEBUG:root:i=209 residual=0.000317238038405776
        DEBUG:root:i=210 residual=0.0003172380675096065
        DEBUG:root:i=211 residual=0.0003172395518049598
        DEBUG:root:i=212 residual=0.00031723809661343694
        DEBUG:root:i=213 residual=0.00031723821302875876
        DEBUG:root:i=214 residual=0.00031723821302875876
        DEBUG:root:i=215 residual=0.0003172380675096065
        DEBUG:root:i=216 residual=0.00031723809661343694
        DEBUG:root:i=217 residual=0.0003172394644934684
        DEBUG:root:i=218 residual=0.0003172381839249283
        DEBUG:root:i=219 residual=0.00031723809661343694
        DEBUG:root:i=220 residual=0.000317238038405776
        DEBUG:root:i=221 residual=0.00031723809661343694
        DEBUG:root:i=222 residual=0.00031723667052574456
        DEBUG:root:i=223 residual=0.00031723809661343694
        DEBUG:root:i=224 residual=0.00031723949359729886
        DEBUG:root:i=225 residual=0.0003139355103485286
        DEBUG:root:i=226 residual=0.0003172365832142532
        DEBUG:root:i=227 residual=0.00031723809661343694
        DEBUG:root:i=228 residual=0.00031723809661343694
        DEBUG:root:i=229 residual=0.00031723809661343694
        DEBUG:root:i=230 residual=0.0003172380675096065
        DEBUG:root:i=231 residual=0.00031723809661343694
        DEBUG:root:i=232 residual=0.00031723943538963795
        DEBUG:root:i=233 residual=0.00031723821302875876
        DEBUG:root:i=234 residual=0.00031723815482109785
        DEBUG:root:i=235 residual=0.00031723809661343694
        DEBUG:root:i=236 residual=0.0003139355103485286
        DEBUG:root:i=237 residual=0.0003205393732059747
        DEBUG:root:i=238 residual=0.00031724091968499124
        DEBUG:root:i=239 residual=0.00031723809661343694
        DEBUG:root:i=240 residual=0.0003172381839249283
        DEBUG:root:i=241 residual=0.000317236699629575
        DEBUG:root:i=242 residual=0.000317238038405776
        DEBUG:root:i=243 residual=0.00031723809661343694
        DEBUG:root:i=244 residual=0.00031723815482109785
        DEBUG:root:i=245 residual=0.0003172395227011293
        DEBUG:root:i=246 residual=0.00031723809661343694
        DEBUG:root:i=247 residual=0.0003172381257172674
        DEBUG:root:i=248 residual=0.00031723800930194557
        DEBUG:root:i=249 residual=0.00031723809661343694
        DEBUG:root:i=250 residual=0.000317238038405776
        DEBUG:root:i=251 residual=0.00031723949359729886
        DEBUG:root:i=252 residual=0.00031723809661343694
        DEBUG:root:i=253 residual=0.0003172381839249283
        DEBUG:root:i=254 residual=0.0003172381839249283
        DEBUG:root:i=255 residual=0.000317238038405776
        DEBUG:root:i=256 residual=0.0003172381839249283
        DEBUG:root:i=257 residual=0.0003172380675096065
        DEBUG:root:i=258 residual=0.00031723949359729886
        DEBUG:root:i=259 residual=0.00031723809661343694
        DEBUG:root:i=260 residual=0.00031723809661343694
        DEBUG:root:i=261 residual=0.00031723667052574456
        DEBUG:root:i=262 residual=0.0003139355103485286
        DEBUG:root:i=263 residual=0.00031723672873340547
        DEBUG:root:i=264 residual=0.000317238038405776
        DEBUG:root:i=265 residual=0.000317238038405776
        DEBUG:root:i=266 residual=0.00031723949359729886
        DEBUG:root:i=267 residual=0.0003172381839249283
        DEBUG:root:i=268 residual=0.0003172380675096065
        DEBUG:root:i=269 residual=0.0003172381839249283
        DEBUG:root:i=270 residual=0.000317238038405776
        DEBUG:root:i=271 residual=0.0003205407701898366
        DEBUG:root:i=272 residual=0.0003172408905811608
        DEBUG:root:i=273 residual=0.0003172382712364197
        DEBUG:root:i=274 residual=0.0003172382421325892
        DEBUG:root:i=275 residual=0.00031723809661343694
        DEBUG:root:i=276 residual=0.000317238038405776
        DEBUG:root:i=277 residual=0.00031723809661343694
        DEBUG:root:i=278 residual=0.0003172395227011293
        DEBUG:root:i=279 residual=0.00031393542303703725
        DEBUG:root:i=280 residual=0.0003172353026457131
        DEBUG:root:i=281 residual=0.000317238038405776
        DEBUG:root:i=282 residual=0.000317238038405776
        DEBUG:root:i=283 residual=0.00031723809661343694
        DEBUG:root:i=284 residual=0.00031723809661343694
        DEBUG:root:i=285 residual=0.00031723809661343694
        DEBUG:root:i=286 residual=0.0003172395518049598
        DEBUG:root:i=287 residual=0.00031723809661343694
        DEBUG:root:i=288 residual=0.000317238038405776
        DEBUG:root:i=289 residual=0.00031723809661343694
        DEBUG:root:i=290 residual=0.00031723809661343694
        DEBUG:root:i=291 residual=0.00031723815482109785
        DEBUG:root:i=292 residual=0.0003139368782285601
        DEBUG:root:i=293 residual=0.00031723533174954355
        DEBUG:root:i=294 residual=0.00031723949359729886
        DEBUG:root:i=295 residual=0.0003172380675096065
        DEBUG:root:i=296 residual=0.0003172381839249283
        DEBUG:root:i=297 residual=0.00031723809661343694
        DEBUG:root:i=298 residual=0.0003139339096378535
        DEBUG:root:i=299 residual=0.0003139339678455144
        DEBUG:root:i=300 residual=0.00031393408426083624
        DEBUG:root:i=301 residual=0.00031723667052574456
        DEBUG:root:i=302 residual=0.00031723809661343694
        DEBUG:root:i=303 residual=0.00031723809661343694
        DEBUG:root:i=304 residual=0.0003172379801981151
        DEBUG:root:i=305 residual=0.0003172394644934684
        DEBUG:root:i=306 residual=0.00031723821302875876
        DEBUG:root:i=307 residual=0.0003205407701898366
        DEBUG:root:i=308 residual=0.00032054216717369854
        DEBUG:root:i=309 residual=0.00032054356415756047
        DEBUG:root:i=310 residual=0.00032054222538135946
        DEBUG:root:i=311 residual=0.0003205423417966813
        DEBUG:root:i=312 residual=0.000317241094307974
        DEBUG:root:i=313 residual=0.0003172381257172674
        DEBUG:root:i=314 residual=0.0003172382421325892
        DEBUG:root:i=315 residual=0.0003205407701898366
        DEBUG:root:i=316 residual=0.0003172408614773303
        DEBUG:root:i=317 residual=0.00031723821302875876
        DEBUG:root:i=318 residual=0.00031723809661343694
        DEBUG:root:i=319 residual=0.00031723809661343694
        DEBUG:root:i=320 residual=0.00031723809661343694
        DEBUG:root:i=321 residual=0.00031723809661343694
        DEBUG:root:i=322 residual=0.0003172380675096065
        DEBUG:root:i=323 residual=0.00031723809661343694
        DEBUG:root:i=324 residual=0.00031723809661343694
        DEBUG:root:i=325 residual=0.00031723809661343694
        DEBUG:root:i=326 residual=0.00031723809661343694
        DEBUG:root:i=327 residual=0.0003172380675096065
        DEBUG:root:i=328 residual=0.0003139368200208992
        DEBUG:root:i=329 residual=0.00031393265817314386
        DEBUG:root:i=330 residual=0.00031723675783723593
        DEBUG:root:i=331 residual=0.0003205421380698681
        DEBUG:root:i=332 residual=0.00032054216717369854
        DEBUG:root:i=333 residual=0.00032054216717369854
        DEBUG:root:i=334 residual=0.00031724103610031307
        DEBUG:root:i=335 residual=0.00031723821302875876
        DEBUG:root:i=336 residual=0.00031723809661343694
        DEBUG:root:i=337 residual=0.00031723815482109785
        DEBUG:root:i=338 residual=0.00031723809661343694
        DEBUG:root:i=339 residual=0.00031393536482937634
        DEBUG:root:i=340 residual=0.0003139339969493449
        DEBUG:root:i=341 residual=0.0003172353026457131
        DEBUG:root:i=342 residual=0.000317238038405776
        DEBUG:root:i=343 residual=0.00031723943538963795
        DEBUG:root:i=344 residual=0.00031723809661343694
        DEBUG:root:i=345 residual=0.00031723809661343694
        DEBUG:root:i=346 residual=0.00031723809661343694
        DEBUG:root:i=347 residual=0.00031723809661343694
        DEBUG:root:i=348 residual=0.0003172380675096065
        DEBUG:root:i=349 residual=0.0003172381257172674
        DEBUG:root:i=350 residual=0.0003172395518049598
        DEBUG:root:i=351 residual=0.0003172380675096065
        DEBUG:root:i=352 residual=0.00031723821302875876
        DEBUG:root:i=353 residual=0.00031723809661343694
        DEBUG:root:i=354 residual=0.0003139353939332068
        DEBUG:root:i=355 residual=0.000317236699629575
        DEBUG:root:i=356 residual=0.000317238038405776
        DEBUG:root:i=357 residual=0.0003172381257172674
        DEBUG:root:i=358 residual=0.000317238038405776
        DEBUG:root:i=359 residual=0.0003172381839249283
        DEBUG:root:i=360 residual=0.0003172380675096065
        DEBUG:root:i=361 residual=0.00031723809661343694
        DEBUG:root:i=362 residual=0.00031723809661343694
        DEBUG:root:i=363 residual=0.0003172381839249283
        DEBUG:root:i=364 residual=0.0003205422544851899
        DEBUG:root:i=365 residual=0.0003172394062858075
        DEBUG:root:i=366 residual=0.0003172381839249283
        DEBUG:root:i=367 residual=0.0003172381257172674
        DEBUG:root:i=368 residual=0.00031723809661343694
        DEBUG:root:i=369 residual=0.0003172395518049598
        DEBUG:root:i=370 residual=0.0003172381839249283
        DEBUG:root:i=371 residual=0.0003172380675096065
        DEBUG:root:i=372 residual=0.00031393536482937634
        DEBUG:root:i=373 residual=0.00031723672873340547
        DEBUG:root:i=374 residual=0.00031723809661343694
        DEBUG:root:i=375 residual=0.00031723809661343694
        DEBUG:root:i=376 residual=0.00031723800930194557
        DEBUG:root:i=377 residual=0.0003172395227011293
        DEBUG:root:i=378 residual=0.00031723809661343694
        DEBUG:root:i=379 residual=0.0003172381839249283
        DEBUG:root:i=380 residual=0.00031723667052574456
        DEBUG:root:i=381 residual=0.00031723809661343694
        DEBUG:root:i=382 residual=0.000317238038405776
        DEBUG:root:i=383 residual=0.00031723809661343694
        DEBUG:root:i=384 residual=0.00031723949359729886
        DEBUG:root:i=385 residual=0.0003172381839249283
        DEBUG:root:i=386 residual=0.0003172381839249283
        DEBUG:root:i=387 residual=0.0003139353939332068
        DEBUG:root:i=388 residual=0.000317236699629575
        DEBUG:root:i=389 residual=0.0003172380675096065
        DEBUG:root:i=390 residual=0.0003139353939332068
        DEBUG:root:i=391 residual=0.0003139339969493449
        DEBUG:root:i=392 residual=0.000317236699629575
        DEBUG:root:i=393 residual=0.000317238038405776
        DEBUG:root:i=394 residual=0.00031393536482937634
        DEBUG:root:i=395 residual=0.0003172353026457131
        DEBUG:root:i=396 residual=0.00031723943538963795
        DEBUG:root:i=397 residual=0.00031723809661343694
        DEBUG:root:i=398 residual=0.00031723809661343694
        DEBUG:root:i=399 residual=0.0003172381257172674
        DEBUG:root:i=400 residual=0.000317238038405776
        DEBUG:root:i=401 residual=0.0003172381839249283
        DEBUG:root:i=402 residual=0.0003172380675096065
        DEBUG:root:i=403 residual=0.00031723949359729886
        DEBUG:root:i=404 residual=0.000317238038405776
        DEBUG:root:i=405 residual=0.0003172381839249283
        DEBUG:root:i=406 residual=0.0003172380675096065
        DEBUG:root:i=407 residual=0.00031723815482109785
        DEBUG:root:i=408 residual=0.00031723821302875876
        DEBUG:root:i=409 residual=0.000317238038405776
        DEBUG:root:i=410 residual=0.00031723949359729886
        DEBUG:root:i=411 residual=0.0003172381839249283
        DEBUG:root:i=412 residual=0.0003172381839249283
        DEBUG:root:i=413 residual=0.00031723809661343694
        DEBUG:root:i=414 residual=0.000317236699629575
        DEBUG:root:i=415 residual=0.000317238038405776
        DEBUG:root:i=416 residual=0.0003172394644934684
        DEBUG:root:i=417 residual=0.0003172380675096065
        DEBUG:root:i=418 residual=0.00031723821302875876
        DEBUG:root:i=419 residual=0.00031723809661343694
        DEBUG:root:i=420 residual=0.00031723809661343694
        DEBUG:root:i=421 residual=0.0003172380675096065
        DEBUG:root:i=422 residual=0.0003172380675096065
        DEBUG:root:i=423 residual=0.0003172395227011293
        DEBUG:root:i=424 residual=0.00031063274946063757
        DEBUG:root:i=425 residual=0.0003073258849326521
        DEBUG:root:i=426 residual=0.00031062838388606906
        DEBUG:root:i=427 residual=0.00031393111567012966
        DEBUG:root:i=428 residual=0.0003172365832142532
        DEBUG:root:i=429 residual=0.00031393536482937634
        DEBUG:root:i=430 residual=0.00031393408426083624
        DEBUG:root:i=431 residual=0.0003139339096378535
        DEBUG:root:i=432 residual=0.00031723533174954355
        DEBUG:root:i=433 residual=0.00031723949359729886
        DEBUG:root:i=434 residual=0.00031723809661343694
        DEBUG:root:i=435 residual=0.00031723809661343694
        DEBUG:root:i=436 residual=0.00031723809661343694
        DEBUG:root:i=437 residual=0.000317238038405776
        DEBUG:root:i=438 residual=0.000317236699629575
        DEBUG:root:i=439 residual=0.0003172395518049598
        DEBUG:root:i=440 residual=0.0003172380675096065
        DEBUG:root:i=441 residual=0.0003172381839249283
        DEBUG:root:i=442 residual=0.00031393536482937634
        DEBUG:root:i=443 residual=0.00030732867890037596
        DEBUG:root:i=444 residual=0.0003073244006372988
        DEBUG:root:i=445 residual=0.00031062844209372997
        DEBUG:root:i=446 residual=0.0003139325126539916
        DEBUG:root:i=447 residual=0.00031393117387779057
        DEBUG:root:i=448 residual=0.00031393382232636213
        DEBUG:root:i=449 residual=0.00031393393874168396
        DEBUG:root:i=450 residual=0.0003139339678455144
        DEBUG:root:i=451 residual=0.00030732862069271505
        DEBUG:root:i=452 residual=0.00030732594314031303
        DEBUG:root:i=453 residual=0.0003106269577983767
        DEBUG:root:i=454 residual=0.00031393111567012966
        DEBUG:root:i=455 residual=0.00031063120695762336
        DEBUG:root:i=456 residual=0.0003073257685173303
        DEBUG:root:i=457 residual=0.0003073258267249912
        DEBUG:root:i=458 residual=0.00031062561902217567
        DEBUG:root:i=459 residual=0.00031393245444633067
        DEBUG:root:i=460 residual=0.0003139339096378535
        DEBUG:root:i=461 residual=0.00031393393874168396
        DEBUG:root:i=462 residual=0.0003139339969493449
        DEBUG:root:i=463 residual=0.0003205393149983138
        DEBUG:root:i=464 residual=0.00031723800930194557
        DEBUG:root:i=465 residual=0.0003172381257172674
        DEBUG:root:i=466 residual=0.0003205407701898366
        DEBUG:root:i=467 residual=0.0003172408905811608
        DEBUG:root:i=468 residual=0.00031723809661343694
        DEBUG:root:i=469 residual=0.0003172381839249283
        DEBUG:root:i=470 residual=0.00031723815482109785
        DEBUG:root:i=471 residual=0.00031723809661343694
        DEBUG:root:i=472 residual=0.00031723958090879023
        DEBUG:root:i=473 residual=0.0003139353939332068
        DEBUG:root:i=474 residual=0.0003172366414219141
        DEBUG:root:i=475 residual=0.000317238038405776
        DEBUG:root:i=476 residual=0.00031723809661343694
        DEBUG:root:i=477 residual=0.00031723809661343694
        DEBUG:root:i=478 residual=0.0003139354521408677
        DEBUG:root:i=479 residual=0.00031393408426083624
        DEBUG:root:i=480 residual=0.00031063120695762336
        DEBUG:root:i=481 residual=0.00031393259996548295
        DEBUG:root:i=482 residual=0.0003106298972852528
        DEBUG:root:i=483 residual=0.000313931202981621
        DEBUG:root:i=484 residual=0.0003139339096378535
        DEBUG:root:i=485 residual=0.0003172365832142532
        DEBUG:root:i=486 residual=0.0003172380675096065
        DEBUG:root:i=487 residual=0.00031723809661343694
        DEBUG:root:i=488 residual=0.00031723949359729886
        DEBUG:root:i=489 residual=0.00031723809661343694
        DEBUG:root:i=490 residual=0.00031723809661343694
        DEBUG:root:i=491 residual=0.00031723809661343694
        DEBUG:root:i=492 residual=0.00031723809661343694
        DEBUG:root:i=493 residual=0.00031723672873340547
        DEBUG:root:i=494 residual=0.00031723943538963795
        DEBUG:root:i=495 residual=0.00031723809661343694
        DEBUG:root:i=496 residual=0.00031723809661343694
        DEBUG:root:i=497 residual=0.00031723809661343694
        DEBUG:root:i=498 residual=0.0003205407701898366
        DEBUG:root:i=499 residual=0.0003172410069964826
        DEBUG:root:i=500 residual=0.00031723821302875876
        DEBUG:root:i=501 residual=0.0003172380675096065
        DEBUG:root:i=502 residual=0.0003172380675096065
        DEBUG:root:i=503 residual=0.0003139353939332068
        DEBUG:root:i=504 residual=0.000317236699629575
        DEBUG:root:i=505 residual=0.000317238038405776
        DEBUG:root:i=506 residual=0.00031723821302875876
        DEBUG:root:i=507 residual=0.00031723943538963795
        DEBUG:root:i=508 residual=0.0003139355103485286
        DEBUG:root:i=509 residual=0.0003139339969493449
        DEBUG:root:i=510 residual=0.0003106313233729452
        DEBUG:root:i=511 residual=0.0003106270742136985
        DEBUG:root:i=512 residual=0.0003139325126539916
        DEBUG:root:i=513 residual=0.00031393382232636213
        DEBUG:root:i=514 residual=0.00030732867890037596
        DEBUG:root:i=515 residual=0.0003139298059977591
        DEBUG:root:i=516 residual=0.00031723518623039126
        DEBUG:root:i=517 residual=0.000317238038405776
        DEBUG:root:i=518 residual=0.0003139353357255459
        DEBUG:root:i=519 residual=0.0003139340551570058
        DEBUG:root:i=520 residual=0.0003106313233729452
        DEBUG:root:i=521 residual=0.0003106298972852528
        DEBUG:root:i=522 residual=0.00031393254175782204
        DEBUG:root:i=523 residual=0.0003139339096378535
        DEBUG:root:i=524 residual=0.00031063129426911473
        DEBUG:root:i=525 residual=0.00031392983510158956
        DEBUG:root:i=526 residual=0.00031393388053402305
        DEBUG:root:i=527 residual=0.0003172366414219141
        DEBUG:root:i=528 residual=0.0003172379801981151
        DEBUG:root:i=529 residual=0.00031723809661343694
        DEBUG:root:i=530 residual=0.0003172395518049598
        DEBUG:root:i=531 residual=0.0003172381839249283
        DEBUG:root:i=532 residual=0.0003172381839249283
        DEBUG:root:i=533 residual=0.0003172381839249283
        DEBUG:root:i=534 residual=0.0003172380675096065
        DEBUG:root:i=535 residual=0.00031723809661343694
        DEBUG:root:i=536 residual=0.0003139353939332068
        DEBUG:root:i=537 residual=0.0003139327163808048
        DEBUG:root:i=538 residual=0.0003106312360614538
        DEBUG:root:i=539 residual=0.0003073272237088531
        DEBUG:root:i=540 residual=0.00031062704510986805
        DEBUG:root:i=541 residual=0.00031062980997376144
        DEBUG:root:i=542 residual=0.00031393254175782204
        DEBUG:root:i=543 residual=0.0003139325126539916
        DEBUG:root:i=544 residual=0.0003106312651652843
        DEBUG:root:i=545 residual=0.00031393111567012966
        DEBUG:root:i=546 residual=0.00031393393874168396
        DEBUG:root:i=547 residual=0.0003073285915888846
        DEBUG:root:i=548 residual=0.0003073258267249912
        DEBUG:root:i=549 residual=0.0003106270742136985
        DEBUG:root:i=550 residual=0.00031062832567840815
        DEBUG:root:i=551 residual=0.00031393111567012966
        DEBUG:root:i=552 residual=0.0003073285333812237
        DEBUG:root:i=553 residual=0.00030732579762116075
        DEBUG:root:i=554 residual=0.0003139284090138972
        DEBUG:root:i=555 residual=0.0003139338514301926
        DEBUG:root:i=556 residual=0.0003172366414219141
        DEBUG:root:i=557 residual=0.00031723809661343694
        DEBUG:root:i=558 residual=0.000317238038405776
        DEBUG:root:i=559 residual=0.0003172395518049598
        DEBUG:root:i=560 residual=0.000317238038405776
        DEBUG:root:i=561 residual=0.0003139353939332068
        DEBUG:root:i=562 residual=0.00031723672873340547
        DEBUG:root:i=563 residual=0.0003139355103485286
        DEBUG:root:i=564 residual=0.00031063001370057464
        DEBUG:root:i=565 residual=0.00031723518623039126
        DEBUG:root:i=566 residual=0.000317238038405776
        DEBUG:root:i=567 residual=0.000317238038405776
        DEBUG:root:i=568 residual=0.00031063274946063757
        DEBUG:root:i=569 residual=0.00031062992638908327
        DEBUG:root:i=570 residual=0.0003073257685173303
        DEBUG:root:i=571 residual=0.00031393111567012966
        DEBUG:root:i=572 residual=0.00031723518623039126
        DEBUG:root:i=573 residual=0.000317238038405776
        DEBUG:root:i=574 residual=0.00031723809661343694
        DEBUG:root:i=575 residual=0.0003139353939332068
        DEBUG:root:i=576 residual=0.0003106312651652843
        DEBUG:root:i=577 residual=0.00031393259996548295
        DEBUG:root:i=578 residual=0.0003073285915888846
        DEBUG:root:i=579 residual=0.00030732445884495974
        DEBUG:root:i=580 residual=0.0003106270742136985
        DEBUG:root:i=581 residual=0.00031393111567012966
        DEBUG:root:i=582 residual=0.00031063120695762336
        DEBUG:root:i=583 residual=0.0003139325126539916
        DEBUG:root:i=584 residual=0.00030732867890037596
        DEBUG:root:i=585 residual=0.0003073243424296379
        DEBUG:root:i=586 residual=0.00031723245047032833
        DEBUG:root:i=587 residual=0.00031393527751788497
        DEBUG:root:i=588 residual=0.00031723667052574456
        DEBUG:root:i=589 residual=0.0003172379801981151
        DEBUG:root:i=590 residual=0.00031723809661343694
        DEBUG:root:i=591 residual=0.0003172381257172674
        DEBUG:root:i=592 residual=0.00031723949359729886
        DEBUG:root:i=593 residual=0.0003139355103485286
        DEBUG:root:i=594 residual=0.000317236699629575
        DEBUG:root:i=595 residual=0.000317236699629575
        DEBUG:root:i=596 residual=0.0003172379801981151
        DEBUG:root:i=597 residual=0.00031723809661343694
        DEBUG:root:i=598 residual=0.0003139353357255459
        DEBUG:root:i=599 residual=0.0003106313815806061
        DEBUG:root:i=600 residual=0.00031723527354188263
        DEBUG:root:i=601 residual=0.000317238038405776
        DEBUG:root:i=602 residual=0.0003172380675096065
        DEBUG:root:i=603 residual=0.00031723809661343694
        DEBUG:root:i=604 residual=0.00031723943538963795
        DEBUG:root:i=605 residual=0.00031393559766002
        DEBUG:root:i=606 residual=0.00030732867890037596
        DEBUG:root:i=607 residual=0.0003073244297411293
        DEBUG:root:i=608 residual=0.00031062556081451476
        DEBUG:root:i=609 residual=0.0003139324835501611
        DEBUG:root:i=610 residual=0.00031063120695762336
        DEBUG:root:i=611 residual=0.0003139325708616525
        DEBUG:root:i=612 residual=0.00030732719460502267
        DEBUG:root:i=613 residual=0.0003040216688532382
        DEBUG:root:i=614 residual=0.0003106255899183452
        DEBUG:root:i=615 residual=0.00030732707818970084
        DEBUG:root:i=616 residual=0.0003073257685173303
        DEBUG:root:i=617 residual=0.000307322945445776
        DEBUG:root:i=618 residual=0.0003139296895824373
        DEBUG:root:i=619 residual=0.0003172364959027618
        DEBUG:root:i=620 residual=0.00032054210896603763
        DEBUG:root:i=621 residual=0.0003172396100126207
        DEBUG:root:i=622 residual=0.00031723809661343694
        DEBUG:root:i=623 residual=0.00031723809661343694
        DEBUG:root:i=624 residual=0.0003139353939332068
        DEBUG:root:i=625 residual=0.0003172353026457131
        DEBUG:root:i=626 residual=0.000317238038405776
        DEBUG:root:i=627 residual=0.00031723949359729886
        DEBUG:root:i=628 residual=0.0003172381839249283
        DEBUG:root:i=629 residual=0.000317238038405776
        DEBUG:root:i=630 residual=0.00031723821302875876
        DEBUG:root:i=631 residual=0.0003139353939332068
        DEBUG:root:i=632 residual=0.0003172366414219141
        DEBUG:root:i=633 residual=0.00032054074108600616
        DEBUG:root:i=634 residual=0.0003172410069964826
        DEBUG:root:i=635 residual=0.00031723809661343694
        DEBUG:root:i=636 residual=0.00031723809661343694
        DEBUG:root:i=637 residual=0.0003172381257172674
        DEBUG:root:i=638 residual=0.0003172380675096065
        DEBUG:root:i=639 residual=0.00031723809661343694
        DEBUG:root:i=640 residual=0.0003139368200208992
        DEBUG:root:i=641 residual=0.0003172367869410664
        DEBUG:root:i=642 residual=0.0003172381257172674
        DEBUG:root:i=643 residual=0.00031723667052574456
        DEBUG:root:i=644 residual=0.0003139353939332068
        DEBUG:root:i=645 residual=0.00031063129426911473
        DEBUG:root:i=646 residual=0.00031393254175782204
        DEBUG:root:i=647 residual=0.0003073286497965455
        DEBUG:root:i=648 residual=0.00030732437153346837
        DEBUG:root:i=649 residual=0.00031392977689392865
        DEBUG:root:i=650 residual=0.0003172359138261527
        DEBUG:root:i=651 residual=0.0003172399883624166
        DEBUG:root:i=652 residual=0.00031723821302875876
        DEBUG:root:i=653 residual=0.0003172381839249283
        DEBUG:root:i=654 residual=0.00031723821302875876
        DEBUG:root:i=655 residual=0.0003172380675096065
        DEBUG:root:i=656 residual=0.0003172381839249283
        DEBUG:root:i=657 residual=0.00031723809661343694
        DEBUG:root:i=658 residual=0.00031723943538963795
        DEBUG:root:i=659 residual=0.00031393542303703725
        DEBUG:root:i=660 residual=0.0003172367869410664
        DEBUG:root:i=661 residual=0.00031723809661343694
        DEBUG:root:i=662 residual=0.00031723809661343694
        DEBUG:root:i=663 residual=0.0003172380675096065
        DEBUG:root:i=664 residual=0.0003172380675096065
        DEBUG:root:i=665 residual=0.00031723672873340547
        DEBUG:root:i=666 residual=0.00031723943538963795
        DEBUG:root:i=667 residual=0.0003172381839249283
        DEBUG:root:i=668 residual=0.0003172380675096065
        DEBUG:root:i=669 residual=0.0003172380675096065
        DEBUG:root:i=670 residual=0.00031723809661343694
        DEBUG:root:i=671 residual=0.00031723809661343694
        DEBUG:root:i=672 residual=0.00031723809661343694
        DEBUG:root:i=673 residual=0.00031723949359729886
        DEBUG:root:i=674 residual=0.0003139353939332068
        DEBUG:root:i=675 residual=0.0003106313233729452
        DEBUG:root:i=676 residual=0.00031723390566185117
        DEBUG:root:i=677 residual=0.00031393676181323826
        DEBUG:root:i=678 residual=0.0003073273692280054
        DEBUG:root:i=679 residual=0.00030732445884495974
        DEBUG:root:i=680 residual=0.00031062698690220714
        DEBUG:root:i=681 residual=0.0003172351571265608
        DEBUG:root:i=682 residual=0.00032054062467068434
        DEBUG:root:i=683 residual=0.0003172408905811608
        DEBUG:root:i=684 residual=0.0003172381839249283
        DEBUG:root:i=685 residual=0.00031723809661343694
        DEBUG:root:i=686 residual=0.0003172381257172674
        DEBUG:root:i=687 residual=0.0003139355103485286
        DEBUG:root:i=688 residual=0.0003172353026457131
        DEBUG:root:i=689 residual=0.000317238038405776
        DEBUG:root:i=690 residual=0.00031723809661343694
        DEBUG:root:i=691 residual=0.00031723943538963795
        DEBUG:root:i=692 residual=0.00031723809661343694
        DEBUG:root:i=693 residual=0.00031723809661343694
        DEBUG:root:i=694 residual=0.0003172381839249283
        DEBUG:root:i=695 residual=0.00031723809661343694
        DEBUG:root:i=696 residual=0.000317238038405776
        DEBUG:root:i=697 residual=0.0003172394644934684
        DEBUG:root:i=698 residual=0.00031723815482109785
        DEBUG:root:i=699 residual=0.00031723809661343694
        DEBUG:root:i=700 residual=0.00031723809661343694
        DEBUG:root:i=701 residual=0.00031723809661343694
        DEBUG:root:i=702 residual=0.00031723809661343694
        DEBUG:root:i=703 residual=0.00031723809661343694
        DEBUG:root:i=704 residual=0.00031723943538963795
        DEBUG:root:i=705 residual=0.0003172381839249283
        DEBUG:root:i=706 residual=0.0003172381257172674
        DEBUG:root:i=707 residual=0.0003172381257172674
        DEBUG:root:i=708 residual=0.00031723667052574456
        DEBUG:root:i=709 residual=0.00031723809661343694
        DEBUG:root:i=710 residual=0.0003172395227011293
        DEBUG:root:i=711 residual=0.0003139353939332068
        DEBUG:root:i=712 residual=0.00031063141068443656
        DEBUG:root:i=713 residual=0.00031723384745419025
        DEBUG:root:i=714 residual=0.00031723800930194557
        DEBUG:root:i=715 residual=0.0003172395227011293
        DEBUG:root:i=716 residual=0.00031723809661343694
        DEBUG:root:i=717 residual=0.0003172380675096065
        DEBUG:root:i=718 residual=0.0003172380675096065
        DEBUG:root:i=719 residual=0.00031723809661343694
        DEBUG:root:i=720 residual=0.0003172381257172674
        DEBUG:root:i=721 residual=0.0003172395518049598
        DEBUG:root:i=722 residual=0.00031723821302875876
        DEBUG:root:i=723 residual=0.00031723809661343694
        DEBUG:root:i=724 residual=0.00031723661231808364
        DEBUG:root:i=725 residual=0.0003172380675096065
        DEBUG:root:i=726 residual=0.00031723809661343694
        DEBUG:root:i=727 residual=0.00031723809661343694
        DEBUG:root:i=728 residual=0.000313936936436221
        DEBUG:root:i=729 residual=0.0003073287371080369
        DEBUG:root:i=730 residual=0.00030732437153346837
        DEBUG:root:i=731 residual=0.00031062704510986805
        DEBUG:root:i=732 residual=0.00031723518623039126
        DEBUG:root:i=733 residual=0.00031723661231808364
        DEBUG:root:i=734 residual=0.00031723809661343694
        DEBUG:root:i=735 residual=0.0003139367327094078
        DEBUG:root:i=736 residual=0.00031063141068443656
        DEBUG:root:i=737 residual=0.000313931202981621
        DEBUG:root:i=738 residual=0.00030732862069271505
        DEBUG:root:i=739 residual=0.0003073257685173303
        DEBUG:root:i=740 residual=0.00031062704510986805
        DEBUG:root:i=741 residual=0.0003139310865662992
        DEBUG:root:i=742 residual=0.00031063120695762336
        DEBUG:root:i=743 residual=0.000313931202981621
        DEBUG:root:i=744 residual=0.00030732856248505414
        DEBUG:root:i=745 residual=0.0003073257685173303
        DEBUG:root:i=746 residual=0.0003139284090138972
        DEBUG:root:i=747 residual=0.0003139338514301926
        DEBUG:root:i=748 residual=0.0003139339096378535
        DEBUG:root:i=749 residual=0.0003106313233729452
        DEBUG:root:i=750 residual=0.0003172352153342217
        DEBUG:root:i=751 residual=0.000317238038405776
        DEBUG:root:i=752 residual=0.000317236699629575
        DEBUG:root:i=753 residual=0.00031723809661343694
        DEBUG:root:i=754 residual=0.000317238038405776
        DEBUG:root:i=755 residual=0.0003172381839249283
        DEBUG:root:i=756 residual=0.0003172394062858075
        DEBUG:root:i=757 residual=0.00031723815482109785
        DEBUG:root:i=758 residual=0.00031723809661343694
        DEBUG:root:i=759 residual=0.0003172381839249283
        DEBUG:root:i=760 residual=0.0003172381839249283
        DEBUG:root:i=761 residual=0.0003172379801981151
        DEBUG:root:i=762 residual=0.00031393548124469817
        DEBUG:root:i=763 residual=0.0003073286497965455
        DEBUG:root:i=764 residual=0.00030732585582882166
        DEBUG:root:i=765 residual=0.00031062704510986805
        DEBUG:root:i=766 residual=0.00031723384745419025
        DEBUG:root:i=767 residual=0.0003172379219904542
        DEBUG:root:i=768 residual=0.000317238038405776
        DEBUG:root:i=769 residual=0.0003172381839249283
        DEBUG:root:i=770 residual=0.00031723949359729886
        DEBUG:root:i=771 residual=0.0003172381257172674
        DEBUG:root:i=772 residual=0.00031723809661343694
        DEBUG:root:i=773 residual=0.00031723809661343694
        DEBUG:root:i=774 residual=0.00031393542303703725
        DEBUG:root:i=775 residual=0.0003106313233729452
        DEBUG:root:i=776 residual=0.00031393265817314386
        DEBUG:root:i=777 residual=0.00031393254175782204
        DEBUG:root:i=778 residual=0.0003139339096378535
        DEBUG:root:i=779 residual=0.0003172366414219141
        DEBUG:root:i=780 residual=0.0003172380675096065
        DEBUG:root:i=781 residual=0.00031723809661343694
        DEBUG:root:i=782 residual=0.0003172381839249283
        DEBUG:root:i=783 residual=0.0003172379801981151
        DEBUG:root:i=784 residual=0.00031723809661343694
        DEBUG:root:i=785 residual=0.00031723949359729886
        DEBUG:root:i=786 residual=0.00031723821302875876
        DEBUG:root:i=787 residual=0.0003172380675096065
        DEBUG:root:i=788 residual=0.0003172381257172674
        DEBUG:root:i=789 residual=0.0003172381257172674
        DEBUG:root:i=790 residual=0.00031723667052574456
        DEBUG:root:i=791 residual=0.0003172380675096065
        DEBUG:root:i=792 residual=0.0003172395227011293
        DEBUG:root:i=793 residual=0.00031393548124469817
        DEBUG:root:i=794 residual=0.00031063129426911473
        DEBUG:root:i=795 residual=0.0003139312320854515
        DEBUG:root:i=796 residual=0.00031393388053402305
        DEBUG:root:i=797 residual=0.00031723667052574456
        DEBUG:root:i=798 residual=0.000317238038405776
        DEBUG:root:i=799 residual=0.00031723943538963795
        DEBUG:root:i=800 residual=0.00031723815482109785
        DEBUG:root:i=801 residual=0.000317236699629575
        DEBUG:root:i=802 residual=0.0003172380675096065
        DEBUG:root:i=803 residual=0.00031723815482109785
        DEBUG:root:i=804 residual=0.00031723809661343694
        DEBUG:root:i=805 residual=0.000317238038405776
        DEBUG:root:i=806 residual=0.0003139368200208992
        DEBUG:root:i=807 residual=0.0003172354481648654
        DEBUG:root:i=808 residual=0.00032054210896603763
        DEBUG:root:i=809 residual=0.0003172395518049598
        DEBUG:root:i=810 residual=0.00031723809661343694
        DEBUG:root:i=811 residual=0.00031723809661343694
        DEBUG:root:i=812 residual=0.0003172381839249283
        DEBUG:root:i=813 residual=0.00031723949359729886
        DEBUG:root:i=814 residual=0.0003172381839249283
        DEBUG:root:i=815 residual=0.0003172380675096065
        DEBUG:root:i=816 residual=0.000317238038405776
        DEBUG:root:i=817 residual=0.00031723809661343694
        DEBUG:root:i=818 residual=0.00031723809661343694
        DEBUG:root:i=819 residual=0.00031723958090879023
        DEBUG:root:i=820 residual=0.0003172381257172674
        DEBUG:root:i=821 residual=0.0003172367869410664
        DEBUG:root:i=822 residual=0.00031723800930194557
        DEBUG:root:i=823 residual=0.00031723809661343694
        DEBUG:root:i=824 residual=0.00031723809661343694
        DEBUG:root:i=825 residual=0.00031393542303703725
        DEBUG:root:i=826 residual=0.00031723667052574456
        DEBUG:root:i=827 residual=0.00031723943538963795
        DEBUG:root:i=828 residual=0.00031723809661343694
        DEBUG:root:i=829 residual=0.0003172381257172674
        DEBUG:root:i=830 residual=0.0003172381839249283
        DEBUG:root:i=831 residual=0.0003172381257172674
        DEBUG:root:i=832 residual=0.0003172381839249283
        DEBUG:root:i=833 residual=0.0003172379801981151
        DEBUG:root:i=834 residual=0.0003106342046521604
        DEBUG:root:i=835 residual=0.0003139312902931124
        DEBUG:root:i=836 residual=0.00030732862069271505
        DEBUG:root:i=837 residual=0.0003139298059977591
        DEBUG:root:i=838 residual=0.0003073285042773932
        DEBUG:root:i=839 residual=0.0003106270742136985
        DEBUG:root:i=840 residual=0.00031062980997376144
        DEBUG:root:i=841 residual=0.0003139311447739601
        DEBUG:root:i=842 residual=0.00031723518623039126
        DEBUG:root:i=843 residual=0.0003172394062858075
        DEBUG:root:i=844 residual=0.0003172380675096065
        DEBUG:root:i=845 residual=0.0003172381257172674
        DEBUG:root:i=846 residual=0.0003172381257172674
        DEBUG:root:i=847 residual=0.00031723809661343694
        DEBUG:root:i=848 residual=0.00031723809661343694
        DEBUG:root:i=849 residual=0.00031723809661343694
        DEBUG:root:i=850 residual=0.00031723943538963795
        DEBUG:root:i=851 residual=0.0003172381839249283
        DEBUG:root:i=852 residual=0.00031723809661343694
        DEBUG:root:i=853 residual=0.00031723809661343694
        DEBUG:root:i=854 residual=0.0003106327203568071
        DEBUG:root:i=855 residual=0.00031393265817314386
        DEBUG:root:i=856 residual=0.00031062992638908327
        DEBUG:root:i=857 residual=0.00031062980997376144
        DEBUG:root:i=858 residual=0.00031393117387779057
        DEBUG:root:i=859 residual=0.00031723661231808364
        DEBUG:root:i=860 residual=0.0003139353066217154
        DEBUG:root:i=861 residual=0.0003106313233729452
        DEBUG:root:i=862 residual=0.00031393254175782204
        DEBUG:root:i=863 residual=0.00031062992638908327
        DEBUG:root:i=864 residual=0.00031062980997376144
        DEBUG:root:i=865 residual=0.00030732573941349983
        DEBUG:root:i=866 residual=0.0003139311447739601
        DEBUG:root:i=867 residual=0.00031393388053402305
        DEBUG:root:i=868 residual=0.0003073272237088531
        DEBUG:root:i=869 residual=0.0003106270742136985
        DEBUG:root:i=870 residual=0.000310629780869931
        DEBUG:root:i=871 residual=0.00030732573941349983
        DEBUG:root:i=872 residual=0.00031393111567012966
        DEBUG:root:i=873 residual=0.0003172365832142532
        DEBUG:root:i=874 residual=0.000317236699629575
        DEBUG:root:i=875 residual=0.0003172379801981151
        DEBUG:root:i=876 residual=0.00031723809661343694
        DEBUG:root:i=877 residual=0.0003172380675096065
        DEBUG:root:i=878 residual=0.00031723809661343694
        DEBUG:root:i=879 residual=0.0003139368200208992
        DEBUG:root:i=880 residual=0.0003106314688920975
        DEBUG:root:i=881 residual=0.000313931202981621
        DEBUG:root:i=882 residual=0.0003073285915888846
        DEBUG:root:i=883 residual=0.00031062847119756043
        DEBUG:root:i=884 residual=0.00031062704510986805
        DEBUG:root:i=885 residual=0.0003139324835501611
        DEBUG:root:i=886 residual=0.00031393388053402305
        DEBUG:root:i=887 residual=0.0003106312651652843
        DEBUG:root:i=888 residual=0.00031393254175782204
        DEBUG:root:i=889 residual=0.00031393254175782204
        DEBUG:root:i=890 residual=0.00031062986818142235
        DEBUG:root:i=891 residual=0.00031062980997376144
        DEBUG:root:i=892 residual=0.0003106298390775919
        DEBUG:root:i=893 residual=0.00031062847119756043
        DEBUG:root:i=894 residual=0.0003073271655011922
        DEBUG:root:i=895 residual=0.0003073257685173303
        DEBUG:root:i=896 residual=0.00031062561902217567
        DEBUG:root:i=897 residual=0.0003073270490858704
        DEBUG:root:i=898 residual=0.0003139298059977591
        DEBUG:root:i=899 residual=0.0003073285915888846
        DEBUG:root:i=900 residual=0.0003139298059977591
        DEBUG:root:i=901 residual=0.0003073285333812237
        DEBUG:root:i=902 residual=0.00030732437153346837
        DEBUG:root:i=903 residual=0.00031062698690220714
        DEBUG:root:i=904 residual=0.00031393105746246874
        DEBUG:root:i=905 residual=0.00031393393874168396
        DEBUG:root:i=906 residual=0.0003139339096378535
        DEBUG:root:i=907 residual=0.00031063129426911473
        DEBUG:root:i=908 residual=0.00031393259996548295
        DEBUG:root:i=909 residual=0.00031393393874168396
        DEBUG:root:i=910 residual=0.00030732719460502267
        DEBUG:root:i=911 residual=0.00031062704510986805
        DEBUG:root:i=912 residual=0.000310629780869931
        DEBUG:root:i=913 residual=0.00031393111567012966
        DEBUG:root:i=914 residual=0.0003106312651652843
        DEBUG:root:i=915 residual=0.00031062986818142235
        DEBUG:root:i=916 residual=0.0003073257685173303
        DEBUG:root:i=917 residual=0.00031062704510986805
        DEBUG:root:i=918 residual=0.000310629780869931
        DEBUG:root:i=919 residual=0.0003139311447739601
        DEBUG:root:i=920 residual=0.0003106312651652843
        DEBUG:root:i=921 residual=0.0003139311447739601
        DEBUG:root:i=922 residual=0.0003106312360614538
        DEBUG:root:i=923 residual=0.00031062986818142235
        DEBUG:root:i=924 residual=0.0003106284129898995
        DEBUG:root:i=925 residual=0.0003106298972852528
        DEBUG:root:i=926 residual=0.0003139324835501611
        DEBUG:root:i=927 residual=0.0003106298390775919
        DEBUG:root:i=928 residual=0.00031062844209372997
        DEBUG:root:i=929 residual=0.00030732713639736176
        DEBUG:root:i=930 residual=0.00031062852940522134
        DEBUG:root:i=931 residual=0.0003106284129898995
        DEBUG:root:i=932 residual=0.0003106284129898995
        DEBUG:root:i=933 residual=0.00030732713639736176
        DEBUG:root:i=934 residual=0.00031062704510986805
        DEBUG:root:i=935 residual=0.0003139324835501611
        DEBUG:root:i=936 residual=0.0003172365832142532
        DEBUG:root:i=937 residual=0.0003172365832142532
        DEBUG:root:i=938 residual=0.0003172380675096065
        DEBUG:root:i=939 residual=0.0003172380675096065
        DEBUG:root:i=940 residual=0.00031723815482109785
        DEBUG:root:i=941 residual=0.0003172381257172674
        DEBUG:root:i=942 residual=0.0003172395518049598
        DEBUG:root:i=943 residual=0.00031723809661343694
        DEBUG:root:i=944 residual=0.00031723809661343694
        DEBUG:root:i=945 residual=0.00031723809661343694
        DEBUG:root:i=946 residual=0.00031723809661343694
        DEBUG:root:i=947 residual=0.00031393542303703725
        DEBUG:root:i=948 residual=0.00032053940230980515
        DEBUG:root:i=949 residual=0.00031724091968499124
        DEBUG:root:i=950 residual=0.00031723815482109785
        DEBUG:root:i=951 residual=0.0003172381257172674
        DEBUG:root:i=952 residual=0.00031723809661343694
        DEBUG:root:i=953 residual=0.00031723809661343694
        DEBUG:root:i=954 residual=0.000317238038405776
        DEBUG:root:i=955 residual=0.00031723949359729886
        DEBUG:root:i=956 residual=0.0003139354521408677
        DEBUG:root:i=957 residual=0.00031723675783723593
        DEBUG:root:i=958 residual=0.00031723681604489684
        DEBUG:root:i=959 residual=0.0003172379801981151
        DEBUG:root:i=960 residual=0.00031723809661343694
        DEBUG:root:i=961 residual=0.000317238038405776
        DEBUG:root:i=962 residual=0.00031393542303703725
        DEBUG:root:i=963 residual=0.0003139339678455144
        DEBUG:root:i=964 residual=0.0003139339969493449
        DEBUG:root:i=965 residual=0.0003139339096378535
        DEBUG:root:i=966 residual=0.00031063135247677565
        DEBUG:root:i=967 residual=0.0003139325708616525
        DEBUG:root:i=968 residual=0.00031723527354188263
        DEBUG:root:i=969 residual=0.000317238038405776
        DEBUG:root:i=970 residual=0.000317238038405776
        DEBUG:root:i=971 residual=0.0003172380675096065
        DEBUG:root:i=972 residual=0.00031393548124469817
        DEBUG:root:i=973 residual=0.0003106313233729452
        DEBUG:root:i=974 residual=0.00031393259996548295
        DEBUG:root:i=975 residual=0.0003172366414219141
        DEBUG:root:i=976 residual=0.0003139353066217154
        DEBUG:root:i=977 residual=0.00031723667052574456
        DEBUG:root:i=978 residual=0.00031723667052574456
        DEBUG:root:i=979 residual=0.00031723809661343694
        DEBUG:root:i=980 residual=0.00031723815482109785
        DEBUG:root:i=981 residual=0.00031723943538963795
        DEBUG:root:i=982 residual=0.0003172381839249283
        DEBUG:root:i=983 residual=0.000317238038405776
        DEBUG:root:i=984 residual=0.000320540857501328
        DEBUG:root:i=985 residual=0.00031723949359729886
        DEBUG:root:i=986 residual=0.00031393684912472963
        DEBUG:root:i=987 residual=0.00031723527354188263
        DEBUG:root:i=988 residual=0.00031723943538963795
        DEBUG:root:i=989 residual=0.0003172381257172674
        DEBUG:root:i=990 residual=0.0003172382421325892
        DEBUG:root:i=991 residual=0.0003172381839249283
        DEBUG:root:i=992 residual=0.00031723800930194557
        DEBUG:root:i=993 residual=0.00031723809661343694
        DEBUG:root:i=994 residual=0.0003172394644934684
        DEBUG:root:i=995 residual=0.0003172381257172674
        DEBUG:root:i=996 residual=0.00031723809661343694
        DEBUG:root:i=997 residual=0.0003172384458594024
        DEBUG:root:i=998 residual=0.0003139341133646667
        DEBUG:root:i=999 residual=0.0003139339096378535
        INFO:root:rank=0 pagerank=1.0624e+01 url=www.lawfareblog.com/lawfare-job-board
        INFO:root:rank=1 pagerank=1.0624e+01 url=www.lawfareblog.com/masthead
        INFO:root:rank=2 pagerank=1.0624e+01 url=www.lawfareblog.com/litigation-documents-resources-related-travel-ban
        INFO:root:rank=3 pagerank=1.0624e+01 url=www.lawfareblog.com/subscribe-lawfare
        INFO:root:rank=4 pagerank=1.0624e+01 url=www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
        INFO:root:rank=5 pagerank=1.0624e+01 url=www.lawfareblog.com/documents-related-mueller-investigation
        INFO:root:rank=6 pagerank=1.0624e+01 url=www.lawfareblog.com/our-comments-policy
        INFO:root:rank=7 pagerank=1.0624e+01 url=www.lawfareblog.com/upcoming-events
        INFO:root:rank=8 pagerank=1.0624e+01 url=www.lawfareblog.com/topics
        INFO:root:rank=9 pagerank=1.0624e+01 url=www.lawfareblog.com/support-lawfare
        
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2
        DEBUG:root:computing indices
        DEBUG:root:computing values
        DEBUG:root:i=0 residual=5.117237567901611
        DEBUG:root:i=1 residual=2.9804813861846924
        DEBUG:root:i=2 residual=2.4491045475006104
        DEBUG:root:i=3 residual=1.698635220527649
        DEBUG:root:i=4 residual=1.1000990867614746
        DEBUG:root:i=5 residual=0.7297568321228027
        DEBUG:root:i=6 residual=0.514761209487915
        DEBUG:root:i=7 residual=0.3795803487300873
        DEBUG:root:i=8 residual=0.2850838005542755
        DEBUG:root:i=9 residual=0.2157047986984253
        DEBUG:root:i=10 residual=0.16483207046985626
        DEBUG:root:i=11 residual=0.12841403484344482
        DEBUG:root:i=12 residual=0.10304608941078186
        DEBUG:root:i=13 residual=0.08559361845254898
        DEBUG:root:i=14 residual=0.07334396988153458
        DEBUG:root:i=15 residual=0.06423044949769974
        DEBUG:root:i=16 residual=0.05689764395356178
        DEBUG:root:i=17 residual=0.05057094991207123
        DEBUG:root:i=18 residual=0.044861793518066406
        DEBUG:root:i=19 residual=0.03961014002561569
        DEBUG:root:i=20 residual=0.03476380184292793
        DEBUG:root:i=21 residual=0.030314506962895393
        DEBUG:root:i=22 residual=0.02626786194741726
        DEBUG:root:i=23 residual=0.022626224905252457
        DEBUG:root:i=24 residual=0.01938277669250965
        DEBUG:root:i=25 residual=0.016521476209163666
        DEBUG:root:i=26 residual=0.014020247384905815
        DEBUG:root:i=27 residual=0.011850166134536266
        DEBUG:root:i=28 residual=0.00998082384467125
        DEBUG:root:i=29 residual=0.008380168117582798
        DEBUG:root:i=30 residual=0.007017120253294706
        DEBUG:root:i=31 residual=0.005861947312951088
        DEBUG:root:i=32 residual=0.004886546637862921
        DEBUG:root:i=33 residual=0.00406653480604291
        DEBUG:root:i=34 residual=0.0033789058215916157
        DEBUG:root:i=35 residual=0.002803673967719078
        DEBUG:root:i=36 residual=0.0023237913846969604
        DEBUG:root:i=37 residual=0.0019244913710281253
        DEBUG:root:i=38 residual=0.0015928337816148996
        DEBUG:root:i=39 residual=0.0013172643957659602
        DEBUG:root:i=40 residual=0.0010889500845223665
        DEBUG:root:i=41 residual=0.0008999287383630872
        DEBUG:root:i=42 residual=0.0007435237057507038
        DEBUG:root:i=43 residual=0.0006142214988358319
        DEBUG:root:i=44 residual=0.0005074203363619745
        DEBUG:root:i=45 residual=0.000419005926232785
        DEBUG:root:i=46 residual=0.00034628110006451607
        DEBUG:root:i=47 residual=0.00028601399390026927
        DEBUG:root:i=48 residual=0.00023641210282221437
        DEBUG:root:i=49 residual=0.00019538190099410713
        DEBUG:root:i=50 residual=0.0001612970809219405
        DEBUG:root:i=51 residual=0.00013338716235011816
        DEBUG:root:i=52 residual=0.00011045379505958408
        DEBUG:root:i=53 residual=9.132847480941564e-05
        DEBUG:root:i=54 residual=7.552596798632294e-05
        DEBUG:root:i=55 residual=6.261348607949913e-05
        DEBUG:root:i=56 residual=5.184937617741525e-05
        DEBUG:root:i=57 residual=4.296215047361329e-05
        DEBUG:root:i=58 residual=3.5611814382718876e-05
        DEBUG:root:i=59 residual=2.9574126529041678e-05
        DEBUG:root:i=60 residual=2.4380973627557978e-05
        DEBUG:root:i=61 residual=2.02734736376442e-05
        DEBUG:root:i=62 residual=1.6909214537008666e-05
        DEBUG:root:i=63 residual=1.3909829249314498e-05
        DEBUG:root:i=64 residual=1.160780266218353e-05
        DEBUG:root:i=65 residual=9.660861906013452e-06
        DEBUG:root:i=66 residual=8.071858246694319e-06
        DEBUG:root:i=67 residual=6.802133611927275e-06
        DEBUG:root:i=68 residual=5.655035693052923e-06
        DEBUG:root:i=69 residual=4.621172593033407e-06
        DEBUG:root:i=70 residual=3.877404651575489e-06
        DEBUG:root:i=71 residual=3.2103971534525044e-06
        DEBUG:root:i=72 residual=2.699147444218397e-06
        DEBUG:root:i=73 residual=2.4067735466815066e-06
        DEBUG:root:i=74 residual=1.8685715303945472e-06
        DEBUG:root:i=75 residual=1.5895365095275338e-06
        DEBUG:root:i=76 residual=1.3939329619461205e-06
        DEBUG:root:i=77 residual=1.143126951319573e-06
        DEBUG:root:i=78 residual=9.235662901119213e-07
        INFO:root:rank=0 pagerank=4.6091e+00 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
        INFO:root:rank=1 pagerank=2.9867e+00 url=www.lawfareblog.com/livestream-nov-21-impeachment-hearings-0
        INFO:root:rank=2 pagerank=2.9669e+00 url=www.lawfareblog.com/opening-statement-david-holmes
        INFO:root:rank=3 pagerank=2.0173e+00 url=www.lawfareblog.com/senate-examines-threats-homeland
        INFO:root:rank=4 pagerank=1.8769e+00 url=www.lawfareblog.com/what-make-first-day-impeachment-hearings
        INFO:root:rank=5 pagerank=1.8762e+00 url=www.lawfareblog.com/livestream-house-armed-services-committee-hearing-f-35-program
        INFO:root:rank=6 pagerank=1.8693e+00 url=www.lawfareblog.com/whats-house-resolution-impeachment
        INFO:root:rank=7 pagerank=1.7655e+00 url=www.lawfareblog.com/congress-us-policy-toward-syria-and-turkey-overview-recent-hearings
        INFO:root:rank=8 pagerank=1.6807e+00 url=www.lawfareblog.com/summary-david-holmess-deposition-testimony
        INFO:root:rank=9 pagerank=9.8346e-01 url=www.lawfareblog.com/events
        
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2 --alpha=0.99999
        DEBUG:root:computing indices
        DEBUG:root:computing values
        DEBUG:root:i=0 residual=6.019961357116699
        DEBUG:root:i=1 residual=4.125228404998779
        DEBUG:root:i=2 residual=3.987910032272339
        DEBUG:root:i=3 residual=3.2539920806884766
        DEBUG:root:i=4 residual=2.479320526123047
        DEBUG:root:i=5 residual=1.9349069595336914
        DEBUG:root:i=6 residual=1.605706810951233
        DEBUG:root:i=7 residual=1.392957091331482
        DEBUG:root:i=8 residual=1.2307753562927246
        DEBUG:root:i=9 residual=1.0955640077590942
        DEBUG:root:i=10 residual=0.984882652759552
        DEBUG:root:i=11 residual=0.9026736617088318
        DEBUG:root:i=12 residual=0.8521610498428345
        DEBUG:root:i=13 residual=0.8327209949493408
        DEBUG:root:i=14 residual=0.8394469022750854
        DEBUG:root:i=15 residual=0.8648774027824402
        DEBUG:root:i=16 residual=0.9013458490371704
        DEBUG:root:i=17 residual=0.9424775838851929
        DEBUG:root:i=18 residual=0.9836395978927612
        DEBUG:root:i=19 residual=1.0217621326446533
        DEBUG:root:i=20 residual=1.054983139038086
        DEBUG:root:i=21 residual=1.0823076963424683
        DEBUG:root:i=22 residual=1.1033427715301514
        DEBUG:root:i=23 residual=1.1180973052978516
        DEBUG:root:i=24 residual=1.1268420219421387
        DEBUG:root:i=25 residual=1.130018711090088
        DEBUG:root:i=26 residual=1.1281501054763794
        DEBUG:root:i=27 residual=1.121811866760254
        DEBUG:root:i=28 residual=1.1115853786468506
        DEBUG:root:i=29 residual=1.098035216331482
        DEBUG:root:i=30 residual=1.0817033052444458
        DEBUG:root:i=31 residual=1.0630838871002197
        DEBUG:root:i=32 residual=1.0426288843154907
        DEBUG:root:i=33 residual=1.0207531452178955
        DEBUG:root:i=34 residual=0.9978127479553223
        DEBUG:root:i=35 residual=0.9741222858428955
        DEBUG:root:i=36 residual=0.9499595761299133
        DEBUG:root:i=37 residual=0.9255528450012207
        DEBUG:root:i=38 residual=0.9011116623878479
        DEBUG:root:i=39 residual=0.8767876625061035
        DEBUG:root:i=40 residual=0.8527275919914246
        DEBUG:root:i=41 residual=0.8290461897850037
        DEBUG:root:i=42 residual=0.8058307766914368
        DEBUG:root:i=43 residual=0.7831431031227112
        DEBUG:root:i=44 residual=0.7610442638397217
        DEBUG:root:i=45 residual=0.739576518535614
        DEBUG:root:i=46 residual=0.7187604308128357
        DEBUG:root:i=47 residual=0.6986168622970581
        DEBUG:root:i=48 residual=0.6791562438011169
        DEBUG:root:i=49 residual=0.6603758931159973
        DEBUG:root:i=50 residual=0.6422728896141052
        DEBUG:root:i=51 residual=0.6248358488082886
        DEBUG:root:i=52 residual=0.6080604195594788
        DEBUG:root:i=53 residual=0.5919231176376343
        DEBUG:root:i=54 residual=0.5764096975326538
        DEBUG:root:i=55 residual=0.5614954233169556
        DEBUG:root:i=56 residual=0.5471670627593994
        DEBUG:root:i=57 residual=0.5333980917930603
        DEBUG:root:i=58 residual=0.5201684832572937
        DEBUG:root:i=59 residual=0.5074545741081238
        DEBUG:root:i=60 residual=0.495239794254303
        DEBUG:root:i=61 residual=0.48349928855895996
        DEBUG:root:i=62 residual=0.472206175327301
        DEBUG:root:i=63 residual=0.46135157346725464
        DEBUG:root:i=64 residual=0.4509105086326599
        DEBUG:root:i=65 residual=0.44086024165153503
        DEBUG:root:i=66 residual=0.43118447065353394
        DEBUG:root:i=67 residual=0.42186954617500305
        DEBUG:root:i=68 residual=0.412893682718277
        DEBUG:root:i=69 residual=0.40424028038978577
        DEBUG:root:i=70 residual=0.39589232206344604
        DEBUG:root:i=71 residual=0.38784125447273254
        DEBUG:root:i=72 residual=0.38007235527038574
        DEBUG:root:i=73 residual=0.37256258726119995
        DEBUG:root:i=74 residual=0.3653167188167572
        DEBUG:root:i=75 residual=0.35831016302108765
        DEBUG:root:i=76 residual=0.3515264093875885
        DEBUG:root:i=77 residual=0.3449670374393463
        DEBUG:root:i=78 residual=0.3386213779449463
        DEBUG:root:i=79 residual=0.3324778974056244
        DEBUG:root:i=80 residual=0.3265075087547302
        DEBUG:root:i=81 residual=0.3207356929779053
        DEBUG:root:i=82 residual=0.3151334524154663
        DEBUG:root:i=83 residual=0.30968931317329407
        DEBUG:root:i=84 residual=0.30440837144851685
        DEBUG:root:i=85 residual=0.2992849349975586
        DEBUG:root:i=86 residual=0.2943054735660553
        DEBUG:root:i=87 residual=0.2894616723060608
        DEBUG:root:i=88 residual=0.28475597500801086
        DEBUG:root:i=89 residual=0.280172199010849
        DEBUG:root:i=90 residual=0.2757151424884796
        DEBUG:root:i=91 residual=0.271369069814682
        DEBUG:root:i=92 residual=0.2671470046043396
        DEBUG:root:i=93 residual=0.26302453875541687
        DEBUG:root:i=94 residual=0.25900936126708984
        DEBUG:root:i=95 residual=0.2550884783267975
        DEBUG:root:i=96 residual=0.2512718439102173
        DEBUG:root:i=97 residual=0.24754397571086884
        DEBUG:root:i=98 residual=0.24390681087970734
        DEBUG:root:i=99 residual=0.24036040902137756
        DEBUG:root:i=100 residual=0.23688353598117828
        DEBUG:root:i=101 residual=0.23350214958190918
        DEBUG:root:i=102 residual=0.23018229007720947
        DEBUG:root:i=103 residual=0.2269420176744461
        DEBUG:root:i=104 residual=0.22377318143844604
        DEBUG:root:i=105 residual=0.22068104147911072
        DEBUG:root:i=106 residual=0.21765248477458954
        DEBUG:root:i=107 residual=0.21467407047748566
        DEBUG:root:i=108 residual=0.2117745727300644
        DEBUG:root:i=109 residual=0.20892782509326935
        DEBUG:root:i=110 residual=0.20614121854305267
        DEBUG:root:i=111 residual=0.20341506600379944
        DEBUG:root:i=112 residual=0.20074385404586792
        DEBUG:root:i=113 residual=0.19811968505382538
        DEBUG:root:i=114 residual=0.19554749131202698
        DEBUG:root:i=115 residual=0.19302217662334442
        DEBUG:root:i=116 residual=0.19055405259132385
        DEBUG:root:i=117 residual=0.1881248950958252
        DEBUG:root:i=118 residual=0.18575012683868408
        DEBUG:root:i=119 residual=0.183406263589859
        DEBUG:root:i=120 residual=0.18111924827098846
        DEBUG:root:i=121 residual=0.17886823415756226
        DEBUG:root:i=122 residual=0.17665301263332367
        DEBUG:root:i=123 residual=0.17448419332504272
        DEBUG:root:i=124 residual=0.17234322428703308
        DEBUG:root:i=125 residual=0.17024850845336914
        DEBUG:root:i=126 residual=0.1681893914937973
        DEBUG:root:i=127 residual=0.16615544259548187
        DEBUG:root:i=128 residual=0.16416756808757782
        DEBUG:root:i=129 residual=0.1622152030467987
        DEBUG:root:i=130 residual=0.16029049456119537
        DEBUG:root:i=131 residual=0.1583934724330902
        DEBUG:root:i=132 residual=0.156526580452919
        DEBUG:root:i=133 residual=0.15470287203788757
        DEBUG:root:i=134 residual=0.15288838744163513
        DEBUG:root:i=135 residual=0.1511169821023941
        DEBUG:root:i=136 residual=0.14937053620815277
        DEBUG:root:i=137 residual=0.14764611423015594
        DEBUG:root:i=138 residual=0.14594915509223938
        DEBUG:root:i=139 residual=0.14428994059562683
        DEBUG:root:i=140 residual=0.14264488220214844
        DEBUG:root:i=141 residual=0.1410273313522339
        DEBUG:root:i=142 residual=0.13943685591220856
        DEBUG:root:i=143 residual=0.13786324858665466
        DEBUG:root:i=144 residual=0.1363116353750229
        DEBUG:root:i=145 residual=0.13478977978229523
        DEBUG:root:i=146 residual=0.13328731060028076
        DEBUG:root:i=147 residual=0.13180667161941528
        DEBUG:root:i=148 residual=0.13034534454345703
        DEBUG:root:i=149 residual=0.12890592217445374
        DEBUG:root:i=150 residual=0.12748035788536072
        DEBUG:root:i=151 residual=0.12608468532562256
        DEBUG:root:i=152 residual=0.12470552325248718
        DEBUG:root:i=153 residual=0.12335334718227386
        DEBUG:root:i=154 residual=0.12200206518173218
        DEBUG:root:i=155 residual=0.12067783623933792
        DEBUG:root:i=156 residual=0.11937528103590012
        DEBUG:root:i=157 residual=0.11808402836322784
        DEBUG:root:i=158 residual=0.11681988835334778
        DEBUG:root:i=159 residual=0.11555368453264236
        DEBUG:root:i=160 residual=0.11432235687971115
        DEBUG:root:i=161 residual=0.11309707909822464
        DEBUG:root:i=162 residual=0.11189606040716171
        DEBUG:root:i=163 residual=0.11070097237825394
        DEBUG:root:i=164 residual=0.10952243208885193
        DEBUG:root:i=165 residual=0.10836263746023178
        DEBUG:root:i=166 residual=0.10721943527460098
        DEBUG:root:i=167 residual=0.1060900166630745
        DEBUG:root:i=168 residual=0.10497166216373444
        DEBUG:root:i=169 residual=0.10386452078819275
        DEBUG:root:i=170 residual=0.10277103632688522
        DEBUG:root:i=171 residual=0.10169922560453415
        DEBUG:root:i=172 residual=0.10064101964235306
        DEBUG:root:i=173 residual=0.09959140419960022
        DEBUG:root:i=174 residual=0.0985502377152443
        DEBUG:root:i=175 residual=0.09752529114484787
        DEBUG:root:i=176 residual=0.09651413559913635
        DEBUG:root:i=177 residual=0.0955113023519516
        DEBUG:root:i=178 residual=0.09452749043703079
        DEBUG:root:i=179 residual=0.09355204552412033
        DEBUG:root:i=180 residual=0.09257978200912476
        DEBUG:root:i=181 residual=0.09162909537553787
        DEBUG:root:i=182 residual=0.09069189429283142
        DEBUG:root:i=183 residual=0.08976319432258606
        DEBUG:root:i=184 residual=0.08884280920028687
        DEBUG:root:i=185 residual=0.08792832493782043
        DEBUG:root:i=186 residual=0.08703255653381348
        DEBUG:root:i=187 residual=0.08613476157188416
        DEBUG:root:i=188 residual=0.0852583646774292
        DEBUG:root:i=189 residual=0.0843929797410965
        DEBUG:root:i=190 residual=0.08353328704833984
        DEBUG:root:i=191 residual=0.08268193900585175
        DEBUG:root:i=192 residual=0.08184684813022614
        DEBUG:root:i=193 residual=0.08101741969585419
        DEBUG:root:i=194 residual=0.08019629120826721
        DEBUG:root:i=195 residual=0.07939136028289795
        DEBUG:root:i=196 residual=0.0785868689417839
        DEBUG:root:i=197 residual=0.07779338210821152
        DEBUG:root:i=198 residual=0.07700551301240921
        DEBUG:root:i=199 residual=0.07623375952243805
        DEBUG:root:i=200 residual=0.07545991241931915
        DEBUG:root:i=201 residual=0.07469961047172546
        DEBUG:root:i=202 residual=0.07395540177822113
        DEBUG:root:i=203 residual=0.07321156561374664
        DEBUG:root:i=204 residual=0.07247866690158844
        DEBUG:root:i=205 residual=0.07175412029027939
        DEBUG:root:i=206 residual=0.07103249430656433
        DEBUG:root:i=207 residual=0.07031918317079544
        DEBUG:root:i=208 residual=0.06961926817893982
        DEBUG:root:i=209 residual=0.06891985237598419
        DEBUG:root:i=210 residual=0.06824169307947159
        DEBUG:root:i=211 residual=0.06755880266427994
        DEBUG:root:i=212 residual=0.06688667833805084
        DEBUG:root:i=213 residual=0.06622274219989777
        DEBUG:root:i=214 residual=0.06556449830532074
        DEBUG:root:i=215 residual=0.064898781478405
        DEBUG:root:i=216 residual=0.06425438821315765
        DEBUG:root:i=217 residual=0.0636182352900505
        DEBUG:root:i=218 residual=0.0629875585436821
        DEBUG:root:i=219 residual=0.06236255168914795
        DEBUG:root:i=220 residual=0.061745744198560715
        DEBUG:root:i=221 residual=0.06114232912659645
        DEBUG:root:i=222 residual=0.06053674966096878
        DEBUG:root:i=223 residual=0.05993668735027313
        DEBUG:root:i=224 residual=0.059339456260204315
        DEBUG:root:i=225 residual=0.058753225952386856
        DEBUG:root:i=226 residual=0.05817250907421112
        DEBUG:root:i=227 residual=0.05759468302130699
        DEBUG:root:i=228 residual=0.05702768266201019
        DEBUG:root:i=229 residual=0.056468818336725235
        DEBUG:root:i=230 residual=0.055907733738422394
        DEBUG:root:i=231 residual=0.0553574301302433
        DEBUG:root:i=232 residual=0.054812632501125336
        DEBUG:root:i=233 residual=0.0542760007083416
        DEBUG:root:i=234 residual=0.05373712629079819
        DEBUG:root:i=235 residual=0.05321158468723297
        DEBUG:root:i=236 residual=0.05268102511763573
        DEBUG:root:i=237 residual=0.05216393619775772
        DEBUG:root:i=238 residual=0.0516497865319252
        DEBUG:root:i=239 residual=0.051154203712940216
        DEBUG:root:i=240 residual=0.05064583569765091
        DEBUG:root:i=241 residual=0.050150878727436066
        DEBUG:root:i=242 residual=0.04966394230723381
        DEBUG:root:i=243 residual=0.04917469993233681
        DEBUG:root:i=244 residual=0.048688411712646484
        DEBUG:root:i=245 residual=0.04821022227406502
        DEBUG:root:i=246 residual=0.04774021729826927
        DEBUG:root:i=247 residual=0.04727307707071304
        DEBUG:root:i=248 residual=0.04680614173412323
        DEBUG:root:i=249 residual=0.046352569013834
        DEBUG:root:i=250 residual=0.04589667171239853
        DEBUG:root:i=251 residual=0.045448869466781616
        DEBUG:root:i=252 residual=0.04500393196940422
        DEBUG:root:i=253 residual=0.04456715285778046
        DEBUG:root:i=254 residual=0.04413321614265442
        DEBUG:root:i=255 residual=0.04370734095573425
        DEBUG:root:i=256 residual=0.043273814022541046
        DEBUG:root:i=257 residual=0.042845819145441055
        DEBUG:root:i=258 residual=0.042433783411979675
        DEBUG:root:i=259 residual=0.042019397020339966
        DEBUG:root:i=260 residual=0.04160783067345619
        DEBUG:root:i=261 residual=0.04119911789894104
        DEBUG:root:i=262 residual=0.04079330712556839
        DEBUG:root:i=263 residual=0.0404033362865448
        DEBUG:root:i=264 residual=0.04000581055879593
        DEBUG:root:i=265 residual=0.039618898183107376
        DEBUG:root:i=266 residual=0.03923220559954643
        DEBUG:root:i=267 residual=0.038851041346788406
        DEBUG:root:i=268 residual=0.03847533091902733
        DEBUG:root:i=269 residual=0.03809726983308792
        DEBUG:root:i=270 residual=0.03772718459367752
        DEBUG:root:i=271 residual=0.03735470771789551
        DEBUG:root:i=272 residual=0.03699290752410889
        DEBUG:root:i=273 residual=0.03663385659456253
        DEBUG:root:i=274 residual=0.03627515956759453
        DEBUG:root:i=275 residual=0.03592444583773613
        DEBUG:root:i=276 residual=0.03557129576802254
        DEBUG:root:i=277 residual=0.03523151949048042
        DEBUG:root:i=278 residual=0.03488403558731079
        DEBUG:root:i=279 residual=0.034549813717603683
        DEBUG:root:i=280 residual=0.0342157818377018
        DEBUG:root:i=281 residual=0.03388457000255585
        DEBUG:root:i=282 residual=0.03355101868510246
        DEBUG:root:i=283 residual=0.03322015330195427
        DEBUG:root:i=284 residual=0.032899968326091766
        DEBUG:root:i=285 residual=0.03258778899908066
        DEBUG:root:i=286 residual=0.03226017951965332
        DEBUG:root:i=287 residual=0.03195104002952576
        DEBUG:root:i=288 residual=0.03163682296872139
        DEBUG:root:i=289 residual=0.031328074634075165
        DEBUG:root:i=290 residual=0.031029898673295975
        DEBUG:root:i=291 residual=0.030729321762919426
        DEBUG:root:i=292 residual=0.030428901314735413
        DEBUG:root:i=293 residual=0.03013906441628933
        DEBUG:root:i=294 residual=0.029846852645277977
        DEBUG:root:i=295 residual=0.029549606144428253
        DEBUG:root:i=296 residual=0.02926553785800934
        DEBUG:root:i=297 residual=0.02898688055574894
        DEBUG:root:i=298 residual=0.0287005752325058
        DEBUG:root:i=299 residual=0.02841966785490513
        DEBUG:root:i=300 residual=0.028154537081718445
        DEBUG:root:i=301 residual=0.0278765931725502
        DEBUG:root:i=302 residual=0.027606595307588577
        DEBUG:root:i=303 residual=0.027334164828062057
        DEBUG:root:i=304 residual=0.027072366327047348
        DEBUG:root:i=305 residual=0.026807986199855804
        DEBUG:root:i=306 residual=0.026546470820903778
        DEBUG:root:i=307 residual=0.026287715882062912
        DEBUG:root:i=308 residual=0.026036975905299187
        DEBUG:root:i=309 residual=0.025788962841033936
        DEBUG:root:i=310 residual=0.025530654937028885
        DEBUG:root:i=311 residual=0.025285571813583374
        DEBUG:root:i=312 residual=0.02504843845963478
        DEBUG:root:i=313 residual=0.024798471480607986
        DEBUG:root:i=314 residual=0.024564281105995178
        DEBUG:root:i=315 residual=0.024319717660546303
        DEBUG:root:i=316 residual=0.024088390171527863
        DEBUG:root:i=317 residual=0.023849397897720337
        DEBUG:root:i=318 residual=0.023628773167729378
        DEBUG:root:i=319 residual=0.02340046688914299
        DEBUG:root:i=320 residual=0.023167109116911888
        DEBUG:root:i=321 residual=0.02294435165822506
        DEBUG:root:i=322 residual=0.02272428199648857
        DEBUG:root:i=323 residual=0.022501785308122635
        DEBUG:root:i=324 residual=0.022287189960479736
        DEBUG:root:i=325 residual=0.022067535668611526
        DEBUG:root:i=326 residual=0.021855873987078667
        DEBUG:root:i=327 residual=0.021641680970788002
        DEBUG:root:i=328 residual=0.021427636966109276
        DEBUG:root:i=329 residual=0.02122158370912075
        DEBUG:root:i=330 residual=0.02102084457874298
        DEBUG:root:i=331 residual=0.020817557349801064
        DEBUG:root:i=332 residual=0.020614514127373695
        DEBUG:root:i=333 residual=0.020414011552929878
        DEBUG:root:i=334 residual=0.020224228501319885
        DEBUG:root:i=335 residual=0.020021485164761543
        DEBUG:root:i=336 residual=0.019831862300634384
        DEBUG:root:i=337 residual=0.01963980495929718
        DEBUG:root:i=338 residual=0.019445238634943962
        DEBUG:root:i=339 residual=0.019253401085734367
        DEBUG:root:i=340 residual=0.019077349454164505
        DEBUG:root:i=341 residual=0.01889093592762947
        DEBUG:root:i=342 residual=0.018707275390625
        DEBUG:root:i=343 residual=0.01853152923285961
        DEBUG:root:i=344 residual=0.018350688740611076
        DEBUG:root:i=345 residual=0.01817253977060318
        DEBUG:root:i=346 residual=0.017986688762903214
        DEBUG:root:i=347 residual=0.01782180927693844
        DEBUG:root:i=348 residual=0.01764661632478237
        DEBUG:root:i=349 residual=0.01747932657599449
        DEBUG:root:i=350 residual=0.01730434224009514
        DEBUG:root:i=351 residual=0.017134588211774826
        DEBUG:root:i=352 residual=0.016980687156319618
        DEBUG:root:i=353 residual=0.016808589920401573
        DEBUG:root:i=354 residual=0.016652286052703857
        DEBUG:root:i=355 residual=0.016490833833813667
        DEBUG:root:i=356 residual=0.016329512000083923
        DEBUG:root:i=357 residual=0.016165634617209435
        DEBUG:root:i=358 residual=0.016012350097298622
        DEBUG:root:i=359 residual=0.01585649885237217
        DEBUG:root:i=360 residual=0.015703322365880013
        DEBUG:root:i=361 residual=0.01555290911346674
        DEBUG:root:i=362 residual=0.015402533113956451
        DEBUG:root:i=363 residual=0.015254905447363853
        DEBUG:root:i=364 residual=0.015115177258849144
        DEBUG:root:i=365 residual=0.014959888532757759
        DEBUG:root:i=366 residual=0.014815119095146656
        DEBUG:root:i=367 residual=0.01466518733650446
        DEBUG:root:i=368 residual=0.014533658511936665
        DEBUG:root:i=369 residual=0.014396972954273224
        DEBUG:root:i=370 residual=0.014249949716031551
        DEBUG:root:i=371 residual=0.014110821299254894
        DEBUG:root:i=372 residual=0.013979585841298103
        DEBUG:root:i=373 residual=0.013845806010067463
        DEBUG:root:i=374 residual=0.013712133280932903
        DEBUG:root:i=375 residual=0.013578592799603939
        DEBUG:root:i=376 residual=0.013445096090435982
        DEBUG:root:i=377 residual=0.013314256444573402
        DEBUG:root:i=378 residual=0.013188783079385757
        DEBUG:root:i=379 residual=0.013065952807664871
        DEBUG:root:i=380 residual=0.012935353443026543
        DEBUG:root:i=381 residual=0.012815317139029503
        DEBUG:root:i=382 residual=0.012684877961874008
        DEBUG:root:i=383 residual=0.012565000914037228
        DEBUG:root:i=384 residual=0.012442572973668575
        DEBUG:root:i=385 residual=0.012320183217525482
        DEBUG:root:i=386 residual=0.012200457975268364
        DEBUG:root:i=387 residual=0.012088694609701633
        DEBUG:root:i=388 residual=0.011971810832619667
        DEBUG:root:i=389 residual=0.011854907497763634
        DEBUG:root:i=390 residual=0.011738104745745659
        DEBUG:root:i=391 residual=0.011631815694272518
        DEBUG:root:i=392 residual=0.01151257287710905
        DEBUG:root:i=393 residual=0.011406444944441319
        DEBUG:root:i=394 residual=0.011295129545032978
        DEBUG:root:i=395 residual=0.011183860711753368
        DEBUG:root:i=396 residual=0.011075315065681934
        DEBUG:root:i=397 residual=0.010969415307044983
        DEBUG:root:i=398 residual=0.010863637551665306
        DEBUG:root:i=399 residual=0.010744825005531311
        DEBUG:root:i=400 residual=0.010652133263647556
        DEBUG:root:i=401 residual=0.010549088940024376
        DEBUG:root:i=402 residual=0.010446149855852127
        DEBUG:root:i=403 residual=0.010348484851419926
        DEBUG:root:i=404 residual=0.010245639830827713
        DEBUG:root:i=405 residual=0.010145430453121662
        DEBUG:root:i=406 residual=0.01004794891923666
        DEBUG:root:i=407 residual=0.009950515814125538
        DEBUG:root:i=408 residual=0.009853177703917027
        DEBUG:root:i=409 residual=0.009758434258401394
        DEBUG:root:i=410 residual=0.009663782082498074
        DEBUG:root:i=411 residual=0.009579627774655819
        DEBUG:root:i=412 residual=0.00947989709675312
        DEBUG:root:i=413 residual=0.009387976489961147
        DEBUG:root:i=414 residual=0.009298785589635372
        DEBUG:root:i=415 residual=0.009201832115650177
        DEBUG:root:i=416 residual=0.00912055466324091
        DEBUG:root:i=417 residual=0.009028908796608448
        DEBUG:root:i=418 residual=0.008947753347456455
        DEBUG:root:i=419 residual=0.008861413225531578
        DEBUG:root:i=420 residual=0.008772562257945538
        DEBUG:root:i=421 residual=0.008686289191246033
        DEBUG:root:i=422 residual=0.008602738380432129
        DEBUG:root:i=423 residual=0.008516655303537846
        DEBUG:root:i=424 residual=0.008433153852820396
        DEBUG:root:i=425 residual=0.008354993537068367
        DEBUG:root:i=426 residual=0.008274233900010586
        DEBUG:root:i=427 residual=0.008196135051548481
        DEBUG:root:i=428 residual=0.008120688609778881
        DEBUG:root:i=429 residual=0.008042720146477222
        DEBUG:root:i=430 residual=0.007964772172272205
        DEBUG:root:i=431 residual=0.007886865176260471
        DEBUG:root:i=432 residual=0.007808977738022804
        DEBUG:root:i=433 residual=0.0077364277094602585
        DEBUG:root:i=434 residual=0.0076559907756745815
        DEBUG:root:i=435 residual=0.007583498954772949
        DEBUG:root:i=436 residual=0.007516256999224424
        DEBUG:root:i=437 residual=0.007441212888807058
        DEBUG:root:i=438 residual=0.007368877995759249
        DEBUG:root:i=439 residual=0.007296497467905283
        DEBUG:root:i=440 residual=0.007226819638162851
        DEBUG:root:i=441 residual=0.007157184183597565
        DEBUG:root:i=442 residual=0.0070850360207259655
        DEBUG:root:i=443 residual=0.007025901693850756
        DEBUG:root:i=444 residual=0.006948571186512709
        DEBUG:root:i=445 residual=0.006886945106089115
        DEBUG:root:i=446 residual=0.006814895663410425
        DEBUG:root:i=447 residual=0.006750782486051321
        DEBUG:root:i=448 residual=0.006691857241094112
        DEBUG:root:i=449 residual=0.006622559856623411
        DEBUG:root:i=450 residual=0.006561127956956625
        DEBUG:root:i=451 residual=0.00649190042167902
        DEBUG:root:i=452 residual=0.006427905987948179
        DEBUG:root:i=453 residual=0.006371854804456234
        DEBUG:root:i=454 residual=0.006310596596449614
        DEBUG:root:i=455 residual=0.006259762682020664
        DEBUG:root:i=456 residual=0.006182877812534571
        DEBUG:root:i=457 residual=0.0061269840225577354
        DEBUG:root:i=458 residual=0.006063221022486687
        DEBUG:root:i=459 residual=0.0060151382349431515
        DEBUG:root:i=460 residual=0.0059514897875487804
        DEBUG:root:i=461 residual=0.005893076304346323
        DEBUG:root:i=462 residual=0.005842532496899366
        DEBUG:root:i=463 residual=0.00578155554831028
        DEBUG:root:i=464 residual=0.0057258945889770985
        DEBUG:root:i=465 residual=0.005672802682965994
        DEBUG:root:i=466 residual=0.005617195274680853
        DEBUG:root:i=467 residual=0.0055668288841843605
        DEBUG:root:i=468 residual=0.005511251278221607
        DEBUG:root:i=469 residual=0.005450486205518246
        DEBUG:root:i=470 residual=0.005402792245149612
        DEBUG:root:i=471 residual=0.005349902901798487
        DEBUG:root:i=472 residual=0.005294492933899164
        DEBUG:root:i=473 residual=0.005244311410933733
        DEBUG:root:i=474 residual=0.0051889242604374886
        DEBUG:root:i=475 residual=0.0051466356962919235
        DEBUG:root:i=476 residual=0.005091316066682339
        DEBUG:root:i=477 residual=0.00504643889144063
        DEBUG:root:i=478 residual=0.004991203546524048
        DEBUG:root:i=479 residual=0.004951599985361099
        DEBUG:root:i=480 residual=0.004901642445474863
        DEBUG:root:i=481 residual=0.004856923129409552
        DEBUG:root:i=482 residual=0.004809611942619085
        DEBUG:root:i=483 residual=0.004767538048326969
        DEBUG:root:i=484 residual=0.0047176494263112545
        DEBUG:root:i=485 residual=0.0046678464859724045
        DEBUG:root:i=486 residual=0.004628442227840424
        DEBUG:root:i=487 residual=0.0045786770060658455
        DEBUG:root:i=488 residual=0.0045393845066428185
        DEBUG:root:i=489 residual=0.004494879860430956
        DEBUG:root:i=490 residual=0.004439943004399538
        DEBUG:root:i=491 residual=0.0044059157371521
        DEBUG:root:i=492 residual=0.0043614632450044155
        DEBUG:root:i=493 residual=0.004324932582676411
        DEBUG:root:i=494 residual=0.004275226965546608
        DEBUG:root:i=495 residual=0.004241331946104765
        DEBUG:root:i=496 residual=0.004189179744571447
        DEBUG:root:i=497 residual=0.004155260976403952
        DEBUG:root:i=498 residual=0.004113595932722092
        DEBUG:root:i=499 residual=0.0040772296488285065
        DEBUG:root:i=500 residual=0.004040814470499754
        DEBUG:root:i=501 residual=0.004001807887107134
        DEBUG:root:i=502 residual=0.003962868358939886
        DEBUG:root:i=503 residual=0.003921282012015581
        DEBUG:root:i=504 residual=0.0038876028265804052
        DEBUG:root:i=505 residual=0.0038408890832215548
        DEBUG:root:i=506 residual=0.003809853922575712
        DEBUG:root:i=507 residual=0.003771008923649788
        DEBUG:root:i=508 residual=0.0037347746547311544
        DEBUG:root:i=509 residual=0.0036985601764172316
        DEBUG:root:i=510 residual=0.003659759880974889
        DEBUG:root:i=511 residual=0.003634087275713682
        DEBUG:root:i=512 residual=0.0035900974180549383
        DEBUG:root:i=513 residual=0.003559222212061286
        DEBUG:root:i=514 residual=0.003525764914229512
        DEBUG:root:i=515 residual=0.0034923083148896694
        DEBUG:root:i=516 residual=0.0034588894341140985
        DEBUG:root:i=517 residual=0.0034228600561618805
        DEBUG:root:i=518 residual=0.003389450255781412
        DEBUG:root:i=519 residual=0.0033612854313105345
        DEBUG:root:i=520 residual=0.0033279024064540863
        DEBUG:root:i=521 residual=0.0032946059945970774
        DEBUG:root:i=522 residual=0.00325603736564517
        DEBUG:root:i=523 residual=0.0032279815059155226
        DEBUG:root:i=524 residual=0.003202493768185377
        DEBUG:root:i=525 residual=0.003166638547554612
        DEBUG:root:i=526 residual=0.0031386290211230516
        DEBUG:root:i=527 residual=0.003108017845079303
        DEBUG:root:i=528 residual=0.003085251897573471
        DEBUG:root:i=529 residual=0.003041605232283473
        DEBUG:root:i=530 residual=0.003018865827471018
        DEBUG:root:i=531 residual=0.002985771279782057
        DEBUG:root:i=532 residual=0.002963046543300152
        DEBUG:root:i=533 residual=0.0029351902194321156
        DEBUG:root:i=534 residual=0.0029020565561950207
        DEBUG:root:i=535 residual=0.0028742137365043163
        DEBUG:root:i=536 residual=0.0028516247402876616
        DEBUG:root:i=537 residual=0.0028132961597293615
        DEBUG:root:i=538 residual=0.0027959428261965513
        DEBUG:root:i=539 residual=0.0027681433130055666
        DEBUG:root:i=540 residual=0.002735140500590205
        DEBUG:root:i=541 residual=0.0027152039110660553
        DEBUG:root:i=542 residual=0.0026822213549166918
        DEBUG:root:i=543 residual=0.0026571599300950766
        DEBUG:root:i=544 residual=0.002626807428896427
        DEBUG:root:i=545 residual=0.002612138632684946
        DEBUG:root:i=546 residual=0.0025897228624671698
        DEBUG:root:i=547 residual=0.0025620197411626577
        DEBUG:root:i=548 residual=0.0025343820452690125
        DEBUG:root:i=549 residual=0.00251460587605834
        DEBUG:root:i=550 residual=0.002489602891728282
        DEBUG:root:i=551 residual=0.0024593211710453033
        DEBUG:root:i=552 residual=0.0024369608145207167
        DEBUG:root:i=553 residual=0.002417216543108225
        DEBUG:root:i=554 residual=0.0023922580294311047
        DEBUG:root:i=555 residual=0.0023725347127765417
        DEBUG:root:i=556 residual=0.0023450495209544897
        DEBUG:root:i=557 residual=0.002317507052794099
        DEBUG:root:i=558 residual=0.0022952004801481962
        DEBUG:root:i=559 residual=0.0022781211882829666
        DEBUG:root:i=560 residual=0.002261123852804303
        DEBUG:root:i=561 residual=0.0022362342569977045
        DEBUG:root:i=562 residual=0.0022114128805696964
        DEBUG:root:i=563 residual=0.0021892243530601263
        DEBUG:root:i=564 residual=0.0021747963037341833
        DEBUG:root:i=565 residual=0.002147397492080927
        DEBUG:root:i=566 residual=0.002127828076481819
        DEBUG:root:i=567 residual=0.0021056667901575565
        DEBUG:root:i=568 residual=0.0020939738024026155
        DEBUG:root:i=569 residual=0.002066587330773473
        DEBUG:root:i=570 residual=0.002047073096036911
        DEBUG:root:i=571 residual=0.002024934161454439
        DEBUG:root:i=572 residual=0.0020158833358436823
        DEBUG:root:i=573 residual=0.001993756275624037
        DEBUG:root:i=574 residual=0.0019690473563969135
        DEBUG:root:i=575 residual=0.0019548400305211544
        DEBUG:root:i=576 residual=0.0019353603711351752
        DEBUG:root:i=577 residual=0.0019132851157337427
        DEBUG:root:i=578 residual=0.001899103750474751
        DEBUG:root:i=579 residual=0.0018796551739796996
        DEBUG:root:i=580 residual=0.0018602577038109303
        DEBUG:root:i=581 residual=0.0018435056554153562
        DEBUG:root:i=582 residual=0.0018240665085613728
        DEBUG:root:i=583 residual=0.0018021059222519398
        DEBUG:root:i=584 residual=0.0017879732185974717
        DEBUG:root:i=585 residual=0.0017712389817461371
        DEBUG:root:i=586 residual=0.0017492922488600016
        DEBUG:root:i=587 residual=0.0017404156969860196
        DEBUG:root:i=588 residual=0.0017184758326038718
        DEBUG:root:i=589 residual=0.001706994022242725
        DEBUG:root:i=590 residual=0.0016850600950419903
        DEBUG:root:i=591 residual=0.0016709824558347464
        DEBUG:root:i=592 residual=0.0016569173894822598
        DEBUG:root:i=593 residual=0.0016323851887136698
        DEBUG:root:i=594 residual=0.0016210172325372696
        DEBUG:root:i=595 residual=0.0016069631092250347
        DEBUG:root:i=596 residual=0.0015955284470692277
        DEBUG:root:i=597 residual=0.001581546850502491
        DEBUG:root:i=598 residual=0.0015622882638126612
        DEBUG:root:i=599 residual=0.0015404942678287625
        DEBUG:root:i=600 residual=0.0015316895442083478
        DEBUG:root:i=601 residual=0.0015177500899881124
        DEBUG:root:i=602 residual=0.0015011231880635023
        DEBUG:root:i=603 residual=0.001489801099523902
        DEBUG:root:i=604 residual=0.0014706396032124758
        DEBUG:root:i=605 residual=0.0014566407771781087
        DEBUG:root:i=606 residual=0.0014453395269811153
        DEBUG:root:i=607 residual=0.0014314186992123723
        DEBUG:root:i=608 residual=0.0014201239682734013
        DEBUG:root:i=609 residual=0.0014088291209191084
        DEBUG:root:i=610 residual=0.0013818703591823578
        DEBUG:root:i=611 residual=0.0013758138520643115
        DEBUG:root:i=612 residual=0.0013593205949291587
        DEBUG:root:i=613 residual=0.0013532766606658697
        DEBUG:root:i=614 residual=0.0013420049799606204
        DEBUG:root:i=615 residual=0.0013307509943842888
        DEBUG:root:i=616 residual=0.0013090478023514152
        DEBUG:root:i=617 residual=0.0012978620361536741
        DEBUG:root:i=618 residual=0.0012866234173998237
        DEBUG:root:i=619 residual=0.0012701592640951276
        DEBUG:root:i=620 residual=0.001264159451238811
        DEBUG:root:i=621 residual=0.0012477627024054527
        DEBUG:root:i=622 residual=0.0012417720863595605
        DEBUG:root:i=623 residual=0.0012306188000366092
        DEBUG:root:i=624 residual=0.001214166753925383
        DEBUG:root:i=625 residual=0.0011978071415796876
        DEBUG:root:i=626 residual=0.0011918279342353344
        DEBUG:root:i=627 residual=0.0011806946713477373
        DEBUG:root:i=628 residual=0.0011747280368581414
        DEBUG:root:i=629 residual=0.0011636066483333707
        DEBUG:root:i=630 residual=0.001147257862612605
        DEBUG:root:i=631 residual=0.0011308547109365463
        DEBUG:root:i=632 residual=0.0011197410058230162
        DEBUG:root:i=633 residual=0.001108629978261888
        DEBUG:root:i=634 residual=0.0011027578730136156
        DEBUG:root:i=635 residual=0.0010915988823398948
        DEBUG:root:i=636 residual=0.0010857259621843696
        DEBUG:root:i=637 residual=0.0010745779145509005
        DEBUG:root:i=638 residual=0.0010634930804371834
        DEBUG:root:i=639 residual=0.0010471839923411608
        DEBUG:root:i=640 residual=0.001036107074469328
        DEBUG:root:i=641 residual=0.001025037607178092
        DEBUG:root:i=642 residual=0.0010244220029562712
        DEBUG:root:i=643 residual=0.0010055205784738064
        DEBUG:root:i=644 residual=0.0009996893350034952
        DEBUG:root:i=645 residual=0.0009886303450912237
        DEBUG:root:i=646 residual=0.0009801944252103567
        DEBUG:root:i=647 residual=0.0009692079620435834
        DEBUG:root:i=648 residual=0.000963393657002598
        DEBUG:root:i=649 residual=0.0009445210453122854
        DEBUG:root:i=650 residual=0.0009439365821890533
        DEBUG:root:i=651 residual=0.0009329690947197378
        DEBUG:root:i=652 residual=0.0009219407802447677
        DEBUG:root:i=653 residual=0.0009161409107036889
        DEBUG:root:i=654 residual=0.0009104119963012636
        DEBUG:root:i=655 residual=0.0008993920055218041
        DEBUG:root:i=656 residual=0.000888379174284637
        DEBUG:root:i=657 residual=0.0008878845837898552
        DEBUG:root:i=658 residual=0.0008664234774187207
        DEBUG:root:i=659 residual=0.0008633258985355496
        DEBUG:root:i=660 residual=0.0008549385820515454
        DEBUG:root:i=661 residual=0.0008492253255099058
        DEBUG:root:i=662 residual=0.0008356207399629056
        DEBUG:root:i=663 residual=0.0008325384696945548
        DEBUG:root:i=664 residual=0.0008320621564052999
        DEBUG:root:i=665 residual=0.0008158631389960647
        DEBUG:root:i=666 residual=0.0008127769688144326
        DEBUG:root:i=667 residual=0.0007991971215233207
        DEBUG:root:i=668 residual=0.0007882789941504598
        DEBUG:root:i=669 residual=0.0007826018263585865
        DEBUG:root:i=670 residual=0.000771702267229557
        DEBUG:root:i=671 residual=0.0007711850921623409
        DEBUG:root:i=672 residual=0.0007576710777357221
        DEBUG:root:i=673 residual=0.0007546260603703558
        DEBUG:root:i=674 residual=0.0007489608833566308
        DEBUG:root:i=675 residual=0.0007406917866319418
        DEBUG:root:i=676 residual=0.0007375813438557088
        DEBUG:root:i=677 residual=0.0007319260621443391
        DEBUG:root:i=678 residual=0.000721048447303474
        DEBUG:root:i=679 residual=0.0007206226582638919
        DEBUG:root:i=680 residual=0.0006992992712184787
        DEBUG:root:i=681 residual=0.0006988751119934022
        DEBUG:root:i=682 residual=0.0006932400283403695
        DEBUG:root:i=683 residual=0.0006849807687103748
        DEBUG:root:i=684 residual=0.0006767427548766136
        DEBUG:root:i=685 residual=0.0006737090297974646
        DEBUG:root:i=686 residual=0.0006654772441834211
        DEBUG:root:i=687 residual=0.0006598421023227274
        DEBUG:root:i=688 residual=0.0006568297394551337
        DEBUG:root:i=689 residual=0.0006460528238676488
        DEBUG:root:i=690 residual=0.0006430509383790195
        DEBUG:root:i=691 residual=0.0006374306976795197
        DEBUG:root:i=692 residual=0.0006291951867751777
        DEBUG:root:i=693 residual=0.000620973005425185
        DEBUG:root:i=694 residual=0.000612750998698175
        DEBUG:root:i=695 residual=0.0006072071264497936
        DEBUG:root:i=696 residual=0.0006042162422090769
        DEBUG:root:i=697 residual=0.0006012203521095216
        DEBUG:root:i=698 residual=0.0005930166807956994
        DEBUG:root:i=699 residual=0.0005848622531630099
        DEBUG:root:i=700 residual=0.0005818882491439581
        DEBUG:root:i=701 residual=0.0005815171170979738
        DEBUG:root:i=702 residual=0.0005655391723848879
        DEBUG:root:i=703 residual=0.0005651740939356387
        DEBUG:root:i=704 residual=0.0005622187745757401
        DEBUG:root:i=705 residual=0.0005593070527538657
        DEBUG:root:i=706 residual=0.0005484907887876034
        DEBUG:root:i=707 residual=0.0005429760785773396
        DEBUG:root:i=708 residual=0.0005347811384126544
        DEBUG:root:i=709 residual=0.000529270269908011
        DEBUG:root:i=710 residual=0.0005236920551396906
        DEBUG:root:i=711 residual=0.0005233515403233469
        DEBUG:root:i=712 residual=0.0005178506835363805
        DEBUG:root:i=713 residual=0.0005070535698905587
        DEBUG:root:i=714 residual=0.0005067787133157253
        DEBUG:root:i=715 residual=0.0005012823385186493
        DEBUG:root:i=716 residual=0.0005009443848393857
        DEBUG:root:i=717 residual=0.0004928519483655691
        DEBUG:root:i=718 residual=0.00048728371621109545
        DEBUG:root:i=719 residual=0.0004870248958468437
        DEBUG:root:i=720 residual=0.0004814712156075984
        DEBUG:root:i=721 residual=0.000473354069981724
        DEBUG:root:i=722 residual=0.0004705019237007946
        DEBUG:root:i=723 residual=0.0004701760190073401
        DEBUG:root:i=724 residual=0.00046470900997519493
        DEBUG:root:i=725 residual=0.0004539939109236002
        DEBUG:root:i=726 residual=0.0004484530072659254
        DEBUG:root:i=727 residual=0.0004455813323147595
        DEBUG:root:i=728 residual=0.00044533636537380517
        DEBUG:root:i=729 residual=0.0004397978773340583
        DEBUG:root:i=730 residual=0.00042909581679850817
        DEBUG:root:i=731 residual=0.00042625542846508324
        DEBUG:root:i=732 residual=0.00042601811583153903
        DEBUG:root:i=733 residual=0.00042308392585255206
        DEBUG:root:i=734 residual=0.00041762407636269927
        DEBUG:root:i=735 residual=0.00041478968341834843
        DEBUG:root:i=736 residual=0.0004119262157473713
        DEBUG:root:i=737 residual=0.0004064759414177388
        DEBUG:root:i=738 residual=0.00040363488369621336
        DEBUG:root:i=739 residual=0.00039811141323298216
        DEBUG:root:i=740 residual=0.00039788606227375567
        DEBUG:root:i=741 residual=0.0003898168506566435
        DEBUG:root:i=742 residual=0.0003895877453032881
        DEBUG:root:i=743 residual=0.0003893731045536697
        DEBUG:root:i=744 residual=0.0003760908148251474
        DEBUG:root:i=745 residual=0.00037587317638099194
        DEBUG:root:i=746 residual=0.00037564727244898677
        DEBUG:root:i=747 residual=0.0003727451548911631
        DEBUG:root:i=748 residual=0.00036469666520133615
        DEBUG:root:i=749 residual=0.000359260942786932
        DEBUG:root:i=750 residual=0.0003590501728467643
        DEBUG:root:i=751 residual=0.0003562160418368876
        DEBUG:root:i=752 residual=0.00034818114363588393
        DEBUG:root:i=753 residual=0.0003479675797279924
        DEBUG:root:i=754 residual=0.00034253738704137504
        DEBUG:root:i=755 residual=0.0003370999766048044
        DEBUG:root:i=756 residual=0.00033689974225126207
        DEBUG:root:i=757 residual=0.00033668766263872385
        DEBUG:root:i=758 residual=0.00033387242001481354
        DEBUG:root:i=759 residual=0.00033366831485182047
        DEBUG:root:i=760 residual=0.00032823655055835843
        DEBUG:root:i=761 residual=0.0003176490718033165
        DEBUG:root:i=762 residual=0.000317500380333513
        DEBUG:root:i=763 residual=0.00031731079798191786
        DEBUG:root:i=764 residual=0.0003144688671454787
        DEBUG:root:i=765 residual=0.000311651878291741
        DEBUG:root:i=766 residual=0.0003114634018857032
        DEBUG:root:i=767 residual=0.0003060436574742198
        DEBUG:root:i=768 residual=0.00030585701460950077
        DEBUG:root:i=769 residual=0.00030044413870200515
        DEBUG:root:i=770 residual=0.00029503009864129126
        DEBUG:root:i=771 residual=0.00029490707674995065
        DEBUG:root:i=772 residual=0.00029473225004039705
        DEBUG:root:i=773 residual=0.0002919436665251851
        DEBUG:root:i=774 residual=0.00028653553454205394
        DEBUG:root:i=775 residual=0.00028113246662542224
        DEBUG:root:i=776 residual=0.000281013228232041
        DEBUG:root:i=777 residual=0.00027561275055631995
        DEBUG:root:i=778 residual=0.0002728677063714713
        DEBUG:root:i=779 residual=0.00027267844416201115
        DEBUG:root:i=780 residual=0.00026727953809313476
        DEBUG:root:i=781 residual=0.0002593010722193867
        DEBUG:root:i=782 residual=0.0002591177762951702
        DEBUG:root:i=783 residual=0.0002589490613900125
        DEBUG:root:i=784 residual=0.00025878215092234313
        DEBUG:root:i=785 residual=0.000256042112596333
        DEBUG:root:i=786 residual=0.00025327911134809256
        DEBUG:root:i=787 residual=0.00025311417994089425
        DEBUG:root:i=788 residual=0.000247716176090762
        DEBUG:root:i=789 residual=0.0002476233639754355
        DEBUG:root:i=790 residual=0.000244821363594383
        DEBUG:root:i=791 residual=0.00023942950065247715
        DEBUG:root:i=792 residual=0.00023939460515975952
        DEBUG:root:i=793 residual=0.00023923507251311094
        DEBUG:root:i=794 residual=0.0002364196116104722
        DEBUG:root:i=795 residual=0.00023108372988644987
        DEBUG:root:i=796 residual=0.00023093316121958196
        DEBUG:root:i=797 residual=0.0002255537110613659
        DEBUG:root:i=798 residual=0.00022545328829437494
        DEBUG:root:i=799 residual=0.00022529684065375477
        DEBUG:root:i=800 residual=0.00022515218006446958
        DEBUG:root:i=801 residual=0.00022242969134822488
        DEBUG:root:i=802 residual=0.00022227557201404124
        DEBUG:root:i=803 residual=0.00021689206187147647
        DEBUG:root:i=804 residual=0.00021681387443095446
        DEBUG:root:i=805 residual=0.00021665765962097794
        DEBUG:root:i=806 residual=0.00020611821673810482
        DEBUG:root:i=807 residual=0.000205961026949808
        DEBUG:root:i=808 residual=0.00020581689022947103
        DEBUG:root:i=809 residual=0.0002057390520349145
        DEBUG:root:i=810 residual=0.0002055929508060217
        DEBUG:root:i=811 residual=0.00020551140187308192
        DEBUG:root:i=812 residual=0.00019753683591261506
        DEBUG:root:i=813 residual=0.00019739006529562175
        DEBUG:root:i=814 residual=0.0001946840638993308
        DEBUG:root:i=815 residual=0.00019454656285233796
        DEBUG:root:i=816 residual=0.00018924013420473784
        DEBUG:root:i=817 residual=0.00018387088493909687
        DEBUG:root:i=818 residual=0.00017857072816696018
        DEBUG:root:i=819 residual=0.0001784278720151633
        DEBUG:root:i=820 residual=0.00017835818289313465
        DEBUG:root:i=821 residual=0.00017821330402512103
        DEBUG:root:i=822 residual=0.00017813389422371984
        DEBUG:root:i=823 residual=0.00017800179193727672
        DEBUG:root:i=824 residual=0.00017793902952689677
        DEBUG:root:i=825 residual=0.00017520230903755873
        DEBUG:root:i=826 residual=0.00017513906641397625
        DEBUG:root:i=827 residual=0.00016976776532828808
        DEBUG:root:i=828 residual=0.00016970213619060814
        DEBUG:root:i=829 residual=0.00016957128536887467
        DEBUG:root:i=830 residual=0.00016951136058196425
        DEBUG:root:i=831 residual=0.00016414090350735933
        DEBUG:root:i=832 residual=0.0001640794362174347
        DEBUG:root:i=833 residual=0.00016395398415625095
        DEBUG:root:i=834 residual=0.0001586574362590909
        DEBUG:root:i=835 residual=0.00015852542128413916
        DEBUG:root:i=836 residual=0.00015582336345687509
        DEBUG:root:i=837 residual=0.000155698973685503
        DEBUG:root:i=838 residual=0.0001556428032927215
        DEBUG:root:i=839 residual=0.00014776560419704765
        DEBUG:root:i=840 residual=0.00014763562649022788
        DEBUG:root:i=841 residual=0.00014493930211756378
        DEBUG:root:i=842 residual=0.00014480766549240798
        DEBUG:root:i=843 residual=0.00014475062198471278
        DEBUG:root:i=844 residual=0.0001446247915737331
        DEBUG:root:i=845 residual=0.000144569028634578
        DEBUG:root:i=846 residual=0.00014451821334660053
        DEBUG:root:i=847 residual=0.00014179896970745176
        DEBUG:root:i=848 residual=0.000139102921821177
        DEBUG:root:i=849 residual=0.00014163150626700372
        DEBUG:root:i=850 residual=0.00014157893019728363
        DEBUG:root:i=851 residual=0.00013888254761695862
        DEBUG:root:i=852 residual=0.00014141699648462236
        DEBUG:root:i=853 residual=0.00013087934348732233
        DEBUG:root:i=854 residual=0.00012812729983124882
        DEBUG:root:i=855 residual=0.0001307153288507834
        DEBUG:root:i=856 residual=0.0001254412782145664
        DEBUG:root:i=857 residual=0.0001279083953704685
        DEBUG:root:i=858 residual=0.00012527238868642598
        DEBUG:root:i=859 residual=0.00012521866301540285
        DEBUG:root:i=860 residual=0.0001276946277357638
        DEBUG:root:i=861 residual=0.0001250606292160228
        DEBUG:root:i=862 residual=0.00012501493620220572
        DEBUG:root:i=863 residual=0.0001274974347325042
        DEBUG:root:i=864 residual=0.00012485857587307692
        DEBUG:root:i=865 residual=0.0001220975536853075
        DEBUG:root:i=866 residual=0.00012204990343889222
        DEBUG:root:i=867 residual=0.0001220085978275165
        DEBUG:root:i=868 residual=0.00012190417328383774
        DEBUG:root:i=869 residual=0.00011402570817153901
        DEBUG:root:i=870 residual=0.00010876075248233974
        DEBUG:root:i=871 residual=0.00010871863196371123
        DEBUG:root:i=872 residual=0.0001059602145687677
        DEBUG:root:i=873 residual=0.00010855540313059464
        DEBUG:root:i=874 residual=0.00010851067781914026
        DEBUG:root:i=875 residual=0.00010575492342468351
        DEBUG:root:i=876 residual=0.00010571386519586667
        DEBUG:root:i=877 residual=9.788201714400202e-05
        DEBUG:root:i=878 residual=0.00010033867874881253
        DEBUG:root:i=879 residual=0.00010029874101746827
        DEBUG:root:i=880 residual=0.00010026012023445219
        DEBUG:root:i=881 residual=0.00010014759754994884
        DEBUG:root:i=882 residual=0.00010010749974753708
        DEBUG:root:i=883 residual=0.00010007378295995295
        DEBUG:root:i=884 residual=0.00010003432544181123
        DEBUG:root:i=885 residual=9.213083831127733e-05
        DEBUG:root:i=886 residual=9.466705523664132e-05
        DEBUG:root:i=887 residual=8.940801490098238e-05
        DEBUG:root:i=888 residual=8.409400470554829e-05
        DEBUG:root:i=889 residual=8.149640052579343e-05
        DEBUG:root:i=890 residual=8.145799074554816e-05
        DEBUG:root:i=891 residual=8.39694039314054e-05
        DEBUG:root:i=892 residual=8.130251080729067e-05
        DEBUG:root:i=893 residual=8.12660000519827e-05
        DEBUG:root:i=894 residual=8.378790516871959e-05
        DEBUG:root:i=895 residual=8.119337871903554e-05
        DEBUG:root:i=896 residual=8.108465408440679e-05
        DEBUG:root:i=897 residual=8.361227810382843e-05
        DEBUG:root:i=898 residual=8.100866398308426e-05
        DEBUG:root:i=899 residual=8.097500540316105e-05
        DEBUG:root:i=900 residual=8.343947411049157e-05
        DEBUG:root:i=901 residual=8.083547436399385e-05
        DEBUG:root:i=902 residual=7.815083517925814e-05
        DEBUG:root:i=903 residual=7.811751129338518e-05
        DEBUG:root:i=904 residual=7.801679748808965e-05
        DEBUG:root:i=905 residual=7.798670412739739e-05
        DEBUG:root:i=906 residual=7.795555575285107e-05
        DEBUG:root:i=907 residual=7.792581163812429e-05
        DEBUG:root:i=908 residual=7.788935181451961e-05
        DEBUG:root:i=909 residual=7.256850221892819e-05
        DEBUG:root:i=910 residual=7.519665086874738e-05
        DEBUG:root:i=911 residual=6.730240420438349e-05
        DEBUG:root:i=912 residual=6.727262370986864e-05
        DEBUG:root:i=913 residual=6.716513598803431e-05
        DEBUG:root:i=914 residual=6.459584255935624e-05
        DEBUG:root:i=915 residual=6.45676554995589e-05
        DEBUG:root:i=916 residual=5.9364083426771685e-05
        DEBUG:root:i=917 residual=5.933313877903856e-05
        DEBUG:root:i=918 residual=6.174894224386662e-05
        DEBUG:root:i=919 residual=5.918302122154273e-05
        DEBUG:root:i=920 residual=5.91529969824478e-05
        DEBUG:root:i=921 residual=6.165194645291194e-05
        DEBUG:root:i=922 residual=5.9093490563100204e-05
        DEBUG:root:i=923 residual=5.898780727875419e-05
        DEBUG:root:i=924 residual=6.14966411376372e-05
        DEBUG:root:i=925 residual=5.893253910471685e-05
        DEBUG:root:i=926 residual=5.8905639889417216e-05
        DEBUG:root:i=927 residual=6.141632184153423e-05
        DEBUG:root:i=928 residual=5.8779023675015196e-05
        DEBUG:root:i=929 residual=5.875332135474309e-05
        DEBUG:root:i=930 residual=6.127601227490231e-05
        DEBUG:root:i=931 residual=5.869611777598038e-05
        DEBUG:root:i=932 residual=5.867062645847909e-05
        DEBUG:root:i=933 residual=5.598057759925723e-05
        DEBUG:root:i=934 residual=5.588545900536701e-05
        DEBUG:root:i=935 residual=5.5862336012069136e-05
        DEBUG:root:i=936 residual=5.583636448136531e-05
        DEBUG:root:i=937 residual=5.581260484177619e-05
        DEBUG:root:i=938 residual=5.579094431595877e-05
        DEBUG:root:i=939 residual=5.576999319600873e-05
        DEBUG:root:i=940 residual=5.567787593463436e-05
        DEBUG:root:i=941 residual=5.044450517743826e-05
        DEBUG:root:i=942 residual=5.309466359904036e-05
        DEBUG:root:i=943 residual=5.307263199938461e-05
        DEBUG:root:i=944 residual=5.0368580559734255e-05
        DEBUG:root:i=945 residual=5.302204954205081e-05
        DEBUG:root:i=946 residual=5.293620779411867e-05
        DEBUG:root:i=947 residual=5.023475387133658e-05
        DEBUG:root:i=948 residual=5.2893694373779e-05
        DEBUG:root:i=949 residual=5.287362728267908e-05
        DEBUG:root:i=950 residual=5.017326839151792e-05
        DEBUG:root:i=951 residual=5.283783684717491e-05
        DEBUG:root:i=952 residual=5.281857738737017e-05
        DEBUG:root:i=953 residual=4.751151573145762e-05
        DEBUG:root:i=954 residual=4.2308474803576246e-05
        DEBUG:root:i=955 residual=4.228959005558863e-05
        DEBUG:root:i=956 residual=4.226444798405282e-05
        DEBUG:root:i=957 residual=3.957446460844949e-05
        DEBUG:root:i=958 residual=4.2224353819619864e-05
        DEBUG:root:i=959 residual=4.220526898279786e-05
        DEBUG:root:i=960 residual=3.942890543839894e-05
        DEBUG:root:i=961 residual=4.208628160995431e-05
        DEBUG:root:i=962 residual=4.2070543713634834e-05
        DEBUG:root:i=963 residual=3.936697612516582e-05
        DEBUG:root:i=964 residual=4.2031540942844003e-05
        DEBUG:root:i=965 residual=4.201561750960536e-05
        DEBUG:root:i=966 residual=3.931129685952328e-05
        DEBUG:root:i=967 residual=4.191348853055388e-05
        DEBUG:root:i=968 residual=4.189635001239367e-05
        DEBUG:root:i=969 residual=3.918410584446974e-05
        DEBUG:root:i=970 residual=4.1861305362544954e-05
        DEBUG:root:i=971 residual=4.184762656223029e-05
        DEBUG:root:i=972 residual=3.913419277523644e-05
        DEBUG:root:i=973 residual=4.181480471743271e-05
        DEBUG:root:i=974 residual=4.179772804491222e-05
        DEBUG:root:i=975 residual=3.9020105759846047e-05
        DEBUG:root:i=976 residual=4.170772444922477e-05
        DEBUG:root:i=977 residual=4.169348540017381e-05
        DEBUG:root:i=978 residual=3.898028808180243e-05
        DEBUG:root:i=979 residual=4.1663963202154264e-05
        DEBUG:root:i=980 residual=4.1649414924904704e-05
        DEBUG:root:i=981 residual=3.892551831086166e-05
        DEBUG:root:i=982 residual=3.89144588552881e-05
        DEBUG:root:i=983 residual=3.6389061278896406e-05
        DEBUG:root:i=984 residual=3.1133942684391513e-05
        DEBUG:root:i=985 residual=3.111424666712992e-05
        DEBUG:root:i=986 residual=3.357672903803177e-05
        DEBUG:root:i=987 residual=3.3562013413757086e-05
        DEBUG:root:i=988 residual=3.354590808157809e-05
        DEBUG:root:i=989 residual=3.353238935233094e-05
        DEBUG:root:i=990 residual=3.3517546398798004e-05
        DEBUG:root:i=991 residual=3.3506436011521146e-05
        DEBUG:root:i=992 residual=3.349142207298428e-05
        DEBUG:root:i=993 residual=3.3477830584160984e-05
        DEBUG:root:i=994 residual=3.339911199873313e-05
        DEBUG:root:i=995 residual=3.0909355700714514e-05
        DEBUG:root:i=996 residual=2.8177975764265284e-05
        DEBUG:root:i=997 residual=3.087389632128179e-05
        DEBUG:root:i=998 residual=3.0866136512486264e-05
        DEBUG:root:i=999 residual=2.813212813634891e-05
        INFO:root:rank=0 pagerank=5.2385e+01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
        INFO:root:rank=1 pagerank=5.2385e+01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
        INFO:root:rank=2 pagerank=7.9438e+00 url=www.lawfareblog.com/cost-using-zero-days
        INFO:root:rank=3 pagerank=2.3700e+00 url=www.lawfareblog.com/lawfare-podcast-former-congressman-brian-baird-and-daniel-schuman-how-congress-can-continue-function
        INFO:root:rank=4 pagerank=1.5529e+00 url=www.lawfareblog.com/events
        INFO:root:rank=5 pagerank=1.1867e+00 url=www.lawfareblog.com/water-wars-increased-us-focus-indo-pacific
        INFO:root:rank=6 pagerank=1.1867e+00 url=www.lawfareblog.com/water-wars-drill-maybe-drill
        INFO:root:rank=7 pagerank=1.1867e+00 url=www.lawfareblog.com/water-wars-disjointed-operations-south-china-sea
        INFO:root:rank=8 pagerank=1.1867e+00 url=www.lawfareblog.com/water-wars-sinking-feeling-philippine-china-relations
        INFO:root:rank=9 pagerank=1.1867e+00 url=www.lawfareblog.com/water-wars-us-china-divide-shangri-la

   ```

   Task 2, part 1:
   ```
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona'
        INFO:root:rank=0 pagerank=8.8870e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
        INFO:root:rank=1 pagerank=8.8867e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
        INFO:root:rank=2 pagerank=1.8256e-01 url=www.lawfareblog.com/chinatalk-how-party-takes-its-propaganda-global
        INFO:root:rank=3 pagerank=1.4907e-01 url=www.lawfareblog.com/rational-security-my-corona-edition
        INFO:root:rank=4 pagerank=1.4907e-01 url=www.lawfareblog.com/brexit-not-immune-coronavirus
        INFO:root:rank=5 pagerank=1.0729e-01 url=www.lawfareblog.com/trump-cant-reopen-country-over-state-objections
        INFO:root:rank=6 pagerank=1.0199e-01 url=www.lawfareblog.com/britains-coronavirus-response
        INFO:root:rank=7 pagerank=1.0199e-01 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
        INFO:root:rank=8 pagerank=9.4298e-02 url=www.lawfareblog.com/lawfare-podcast-mom-and-dad-talk-clinical-trials-pandemic
        INFO:root:rank=9 pagerank=8.7207e-02 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
   ```

   Task 2, part 2:
   ```
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona' --search_query='-corona'
        INFO:root:rank=0 pagerank=8.8870e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
        INFO:root:rank=1 pagerank=8.8867e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
        INFO:root:rank=2 pagerank=1.8256e-01 url=www.lawfareblog.com/chinatalk-how-party-takes-its-propaganda-global
        INFO:root:rank=3 pagerank=1.0729e-01 url=www.lawfareblog.com/trump-cant-reopen-country-over-state-objections
        INFO:root:rank=4 pagerank=9.4298e-02 url=www.lawfareblog.com/lawfare-podcast-mom-and-dad-talk-clinical-trials-pandemic
        INFO:root:rank=5 pagerank=7.9633e-02 url=www.lawfareblog.com/fault-lines-foreign-policy-quarantined
        INFO:root:rank=6 pagerank=7.5307e-02 url=www.lawfareblog.com/limits-world-health-organization
        INFO:root:rank=7 pagerank=6.8115e-02 url=www.lawfareblog.com/chinatalk-dispatches-shanghai-beijing-and-hong-kong
        INFO:root:rank=8 pagerank=6.4847e-02 url=www.lawfareblog.com/us-moves-dismiss-case-against-company-linked-ira-troll-farm
        INFO:root:rank=9 pagerank=6.4847e-02 url=www.lawfareblog.com/livestream-house-armed-services-committee-holds-hearing-priorities-missile-defense
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
