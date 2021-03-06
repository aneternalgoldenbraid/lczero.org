---
title: "Dubslow's analysis of Test 10 problems"
weight: 500
wikiname: "Dubslow's-analysis-of-Test-10-problems"
# Warning: File is automatically generated from GitHub wiki, do not edit by hand.
---
So, the major assumption that goes into this is that the Tensorboard data is a reliable sole source of information for the information it provides. That's quite reasonable (if it wasn't we may as well give up the project), but it's also in contrast to the various Elo measurements we have, none of which are a single sole reliable source of playing strenth -- instead they all need to be considered collectively.

So we'll consider Tensorboard as fairly authorative.

As for my own biases, I'm on the record stating previously during various test runs that I think 1 sample per position ("sampling rate") is, while workable, at the very top end of workable; each position is necessarily highly correlated with its parent and child positions, so we don't really gain any information by sampling every position once as opposed to, say, every other position once. I hear that A0 used a sampling rate near 1, but I'm also pretty certain that earlier papers of theirs mentioned specifically sampling less than once per position, specifically to avoid overfitting caused by sampling so much highly correlated data. So these opinions of mine color my theory a bit, but I think we can test some of this.

The major data points I want to bring up are from Tensorboard, hence the disclaimer about it first. Specifically:

![Tensorboard test-train divergence](https://i.imgur.com/1iUEv2O.png)

Notice that the test-train divergence begins around step 120K; if you check the timestamp, you'll see this is around net 10010 or so. This bears repeating: *the value head has been overfit since close to the beginning of test 10*.

Now recently we implemented the effective-large-batchsize training regimen, which more closely mimics the A0 training setup; I'm told that large batchsizes also have beneficial side benefits relating to variance, though I'm not sure I believe that; at any rate, what is certain is that, barring a bug in the implementation, this is essentially guaranteed to be no worse training than before, and in fact may be better. Regardless, at the same time the LR was also lowered, with this resulting MSE graph:

![LR lowered](https://i.imgur.com/92R7WAN.png)

Note that since the LR drop, the test-train split has been growing. Although it's certain that the existing gap was indicative of overfitting, it's also certain that the overfitting is getting worse, apparently caused by the LR drop. I'm not certain why this could be, but it at least passes the smell test. (Any other SGD experts want to weigh in on this one?) It's worth noting that strength continued to grow until perhaps a 15-20 nets after the LR drop before plateauing and then falling. Also, recently we reduced the sampling rate slightly, though it doesn't appear to have had a significant effect yet.

Now, here are some specific features of the test train split, as viewed since the doubling of games per net, which reduces noise and increases data points per net, making it easier to visually judge:

![Doubling games per net](https://i.imgur.com/7BcWUYg.png)

First, note the training loss behavior. Even if we didn't have test-train seperation (which, thank god we do) pointing to overfitting, the training behavior alone strongly hints at it. As each network begins its training on a window with new data, the training loss spikes massively, before falling precipitously. The initial spike at the beginning is indicative of overfitting: if the value head was properly generalizing, it should have similar valuation on the not-much-different new data relative to the old data. Further, the training loss falling as much or farther than the upward spike also indicates that the value head is memorizing the local positions rather than generalizing upon them; if it were generalizing, the downward trend would otherwise be both smoother and slower.

Meanwhile the test loss seems to be showing opposite behavior, though it's a lot noisier (less data to produce each point, and fewer points as well). Subjectively, I would argue that the test loss is *falling* at the beginning of the window; I'm not precisely sure what this means, but I suspect it means that the network is still properly learning *something* in addition to the overfitting. More obviously telling is how the test loss subjectively increases over the course of each cycle, as the training loss falls -- that also indicates that the value head is overfitting.

So we get it, the value head is overfitting, what's the big deal? Well, my hypothesis is that (1) the overfitting beginning near the start of the test run was in fact a bad thing the whole time, even though we ignored it because strength was growing; (2) the recent problems are only the culmination of the long term underlying problems. This is exactly what vexed the original main run, in that despite the *severe* oversampling that was going on, the network managed to gain strength for a long time, masking the problem. This run in combination with the original main run are thus evidence that overfitting *is always* a problem, even if we seem to be gaining strength in the meantime. Something about the LR drop must have exacerbated the problem to lead to the current problems, but the LR drop wasn't bad in and of itself; same with the old main run and reducing the cpuct, it was just one change too many for an already-too-fragile net.

(3) As for fixing it, well, I suggest we reduce the sampling rate substantially further than before. If reducing the sampling rate helps with the overfitting, then we should see the training spikes and drops indicated in the most recent graph start to flatten and smooth, lose overall magnitude in both directions. If we see such over the next 5-10 nets (8 nets is 24 hours), then we ought to conclude that sampling rate was indeed a major underlying cause, and that such a reduction should help prevent future problems. Furthermore, while reducing the sampling rate and watching tensorboard as such ought to confirm its suitability as a cause and long term fix, it won't actually reduce the test train split itself; to that end, I propose that after the reduced sampling rate test, if it comes back as expected, we then temporarily reduce the value head weight to 0.25, as was done previously on the main net; this ought to restore significant playing strength and reduce or eliminate the test-train split on MSE loss, while further problems would be prevented by the reduced sampling rate.

So what should the sampling rate be? Well, directly after the batchsize training improvement, we had a sampling rate of `4096*1250/(60000*84) = 1.016` samples per position. (84 ply per game is what Tilps recently reported, and I suspect it hasn't changed much over the last several million games.) After the first, slight, reduction in sample rate, we are now at `4096*1250/(80000*84) = 4096*2500/(160000*84) = 0.762` samples per position (the doubling of steps-and-games per net was a no-op regarding sampling rate); I propose we aim for a more drastic reduction to perhaps 0.5 samples per position (while in theory we could go as low or maybe even lower than say 0.1 samples per position, which is to say, one sample for every 10 positions; the ideal would be one sample *per game*, but in practice that's much slower for minimal gain). 2000 steps for every 200 000 games, for example (or 1000 per 100 000) would give a sampling rate of `4096*2000/(200000*84) = 0.488` samples per position, which should, in theory, help quite a bit, as to be determined by watching the training spike-and-fall magnitudes for several nets. After that is confirmed to help, then we can temporarily reduce the value head weight to eliminate the test-train split, and hopefully see much elo gain as happened on the old 3xx main nets.

In particular, I am highly skeptical that puct, LR change, training changes, or resign had anything to do with *causing* the recent stregnth losses, if not necessarily acting as exacerbating factors; I'm rather convinced that, based on that very first graph, test 10 has been overfit for most of its run, and like old main before it, managed to gain most of its current strength *in spite of that overfitting*, and that fixing the overfitting via the above method ought to restore strength and strength gaining.

So, tell me why I'm stupid, go!

Taken from Discord LCZero#general 2018-07-22 05:37 +0200 CAT

https://pastebin.com/raw/cqH9J5k6