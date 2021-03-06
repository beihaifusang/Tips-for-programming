# Introduction to Conditional Random Fields

Imagine you have a sequence of snapshots from a day in Justin Bieber’s life, and you want to label each image with the activity it represents (eating, sleeping, driving, etc.). How can you do this?

One way is to ignore the sequential nature of the snapshots, and build a *per-image* classifier. For example, given a month’s worth of labeled snapshots, you might learn that dark images taken at 6am tend to be about sleeping, images with lots of bright colors tend to be about dancing, images of cars are about driving, and so on.

By ignoring this sequential aspect, however, you lose a lot of information. For example, what happens if you see a close-up picture of a mouth – is it about singing or eating? If you know that the *previous* image is a picture of Justin Bieber eating or cooking, then it’s more likely this picture is about eating; if, however, the previous image contains Justin Bieber singing or dancing, then this one probably shows him singing as well.

Thus, to increase the accuracy of our labeler, we should incorporate the labels of nearby photos, and this is precisely what a **conditional random field** does.

# Part-of-Speech Tagging

Let’s go into some more detail, using the more common example of **part-of-speech tagging**.

In POS tagging, the goal is to label a sentence (a sequence of words or tokens) with tags like ADJECTIVE, NOUN, PREPOSITION, VERB, ADVERB, ARTICLE.

For example, given the sentence “Bob drank coffee at Starbucks”, the labeling might be “Bob (NOUN) drank (VERB) coffee (NOUN) at (PREPOSITION) Starbucks (NOUN)”.

So let’s build a conditional random field to label sentences with their parts of speech. Just like any classifier, we’ll first need to decide on a set of feature functions $f_i$.

## Feature Functions in a CRF

In a CRF, each **feature function** is a function that takes in as input:

- a sentence $s$ 
- the position $i$ of a word in the sentence
- the label $l_i$ of the current word
- the label $l_{i-1}$ of the previous word

and outputs a real-valued number (though the numbers are often just either 0 or 1).

(Note: by restricting our features to depend on only the *current* and *previous* labels, rather than arbitrary labels throughout the sentence, I’m actually building the special case of a **linear-chain CRF**. For simplicity, I’m going to ignore general CRFs in this post.)

For example, one possible feature function could measure how much we suspect that the current word should be labeled as an adjective given that the previous word is “very”.

## Features to Probabilities

Next, assign each feature function $f_j$ a **weight** $\lambda_j$ (I’ll talk below about how to learn these weights from the data). Given a sentence s, we can now score a labeling $l$ given $s$ by adding up the weighted features over all words in the sentence:

$\displaystyle score(l | s) = \sum_{j = 1}^m \sum_{i = 1}^n \lambda_j f_j(s, i, l_i, l_{i-1})$

(The first sum runs over each feature function $j$, and the inner sum runs over each position $i$ of the sentence.)

Finally, we can transform these scores into probabilities $p(l | s)$ between 0 and 1 by exponentiating and normalizing:

$\displaystyle p(l | s) = \frac{\displaystyle exp[score(l|s)]}{\displaystyle\sum_{l’} exp[score(l’|s)]} = \frac{\displaystyle exp[\sum_{j = 1}^m \sum_{i = 1}^n \lambda_j f_j(s, i, l_i, l_{i-1})]}{\displaystyle\sum_{l’} exp[\sum_{j = 1}^m \sum_{i = 1}^n \lambda_j f_j(s, i, l’_i, l’_{i-1})]}$

## Example Feature Functions

So what do these feature functions look like? Examples of POS tagging features could include:

- $f_1(s, i, l_i, l_{i-1}) = 1$ if $l_i =$ ADVERB and the ith word ends in “-ly”; 0 otherwise.
  - If the weight $\lambda_1$ associated with this feature is large and positive, then this feature is essentially saying that we prefer labelings where words ending in -ly get labeled as ADVERB.
- $f_2(s, i, l_i, l_{i-1}) = 1$ if $i = 1$, $l_i =$ VERB, and the sentence ends in a question mark; 0 otherwise.
  - Again, if the weight $\lambda_2$ associated with this feature is large and positive, then labelings that assign VERB to the first word in a question (e.g., “Is this a sentence beginning with a verb?”) are preferred.
- $f_3(s, i, l_i, l_{i-1}) = 1$ if $l_{i-1} =$ ADJECTIVE and $l_i =$ NOUN; 0 otherwise.
  - Again, a positive weight for this feature means that adjectives tend to be followed by nouns.
- $f_4(s, i, l_i, l_{i-1}) = 1$ if $l_{i-1} =$ PREPOSITION and $l_i =$ PREPOSITION.
  - A *negative* weight $\lambda_4$ for this function would mean that prepositions don’t tend to follow prepositions, so we should avoid labelings where this happens.

And that’s it! To sum up: to build a conditional random field, you just define a bunch of feature functions (which can depend on the entire sentence, a current position, and nearby labels), assign them weights, and add them all together, transforming at the end to a probability if necessary.

Now let’s step back and compare CRFs to some other common machine learning techniques.

# Smells like Logistic Regression…

The form of the CRF probabilities $\displaystyle p(l | s) = \frac{\displaystyle exp[\sum_{j = 1}^m \sum_{i = 1}^n f_j(s, i, l_i, l_{i-1})]}{\displaystyle \sum_{l’} exp[\sum_{j = 1}^m \sum_{i = 1}^n f_j(s, i, l’_i, l’_{i-1})]}$ might look [familiar](http://en.wikipedia.org/wiki/Logistic_regression).

That’s because CRFs are indeed basically the sequential version of **logistic regression**: whereas logistic regression is a log-linear model for *classification*, CRFs are a log-linear model for *sequential labels*.

# Looks like HMMs…

Recall that **Hidden Markov Models** are another model for part-of-speech tagging (and sequential labeling in general). Whereas CRFs throw any bunch of functions together to get a label score, HMMs take a *generative* approach to labeling, defining

$\displaystyle p(l,s) = p(l_1) \prod_i p(l_i | l_{i-1}) p(w_i | l_i)$

where

- $p(l_i | l_{i-1})$ are **transition** probabilities (e.g., the probability that a preposition is followed by a noun);
- $p(w_i | l_i)$ are **emission** probabilities (e.g., the probability that a noun emits the word “dad”).

So how do HMMs compare to CRFs? CRFs are more powerful – they can model everything HMMs can and more. One way of seeing this is as follows.

Note that the log of the HMM probability is $\displaystyle \log p(l,s) = \log p(l_0) + \sum_i \log p(l_i | l_{i-1}) + \sum_i \log p(w_i | l_i)$. This has exactly the log-linear form of a CRF if we consider these log-probabilities to be the weights associated to binary transition and emission indicator features.

That is, we can build a CRF equivalent to any HMM by…

- For each HMM *transition* probability $ p(l_i = y | l_{i-1} = x) $, define a set of CRF transition features of the form $f_{x,y}(s, i, l_i, l_{i-1}) = 1$ if $l_i = y$ and $l_{i-1} = x$. Give each feature a weight of $w_{x,y} = \log p(l_i = y | l_{i-1} = x)$.
- Similarly, for each HMM *emission* probability $p(w_i = z | l_{i} = x)$, define a set of CRF emission features of the form $g_{x,y}(s, i, l_i, l_{i-1}) = 1$ if $w_i = z$ and $l_i = x$. Give each feature a weight of $w_{x,z} = \log p(w_i = z | l_i = x)$.

Thus, the score $p(l|s)$ computed by a CRF using these feature functions is precisely proportional to the score computed by the associated HMM, and so every HMM is equivalent to some CRF.

However, CRFs can model a much richer set of label distributions as well, for two main reasons:

- **CRFs can define a much larger set of features.** Whereas HMMs are necessarily *local* in nature (because they’re constrained to binary transition and emission feature functions, which force each word to depend only on the current label and each label to depend only on the previous label), CRFs can use more *global* features. For example, one of the features in our POS tagger above increased the probability of labelings that tagged the *first* word of a sentence as a VERB if the *end* of the sentence contained a question mark.
- **CRFs can have arbitrary weights.** Whereas the probabilities of an HMM must satisfy certain constraints (e.g., $\displaystyle 0 \leq p(w_i | l_i) \leq 1, \sum_w p(w_i = w | l_1) = 1)$, the weights of a CRF are unrestricted (e.g., $\log p(w_i | l_i)$ can be anything it wants).

# Learning Weights

Let’s go back to the question of how to learn the feature weights in a CRF. One way is (surprise) to use **gradient ascent**.

Assume we have a bunch of training examples (sentences and associated part-of-speech labels). Randomly initialize the weights of our CRF model. To shift these randomly initialized weights to the correct ones, for each training example…

- Go through each feature function $f_i$, and calculate the gradient of the log probability of the training example with respect to $\lambda_i$: $\displaystyle\frac{\partial}{\partial w_j} \log p(l | s) = \sum_{j = 1}^m f_i(s, j, l_j, l_{j-1}) - \sum_{l’} p(l’ | s) \sum_{j = 1}^m f_i(s, j, l’_j, l’_{j-1})$
- Note that the first term in the gradient is the contribution of feature $f_i$ under the *true* label, and the second term in the gradient is the *expected* contribution of feature $f_i$ under the current model. This is exactly the form you’d expect gradient ascent to take.
- Move $\lambda_i$ in the direction of the gradient: $\displaystyle \lambda_i = \lambda_i + \alpha [\sum_{j = 1}^m f_i(s, j, l_j, l_{j-1}) - \sum_{l’} p(l’ | s) \sum_{j = 1}^m f_i(s, j, l’_j, l’_{j-1})]$ where $\alpha$ is some learning rate.
- Repeat the previous steps until some stopping condition is reached (e.g., the updates fall below some threshold).

In other words, every step takes the difference between what we want the model to learn and the model’s current state, and moves $\lambda_i$ in the direction of this difference.

# Finding the Optimal Labeling

Suppose we’ve trained our CRF model, and now a new sentence comes in. How do we do label it?

The naive way is to calculate $p(l | s)$ for every possible labeling l, and then choose the label that maximizes this probability. However, since there are $k^m$ possible labels for a tag set of size k and a sentence of length m, this approach would have to check an exponential number of labels.

A better way is to realize that (linear-chain) CRFs satisfy an [optimal substructure](http://en.wikipedia.org/wiki/Optimal_substructure) property that allows us to use a (polynomial-time) dynamic programming algorithm to find the optimal label, similar to the [Viterbi algorithm](http://en.wikipedia.org/wiki/Viterbi_algorithm) for HMMs.

# A More Interesting Application

Okay, so part-of-speech tagging is kind of boring, and there are plenty of existing POS taggers out there. When might you use a CRF in real life?

Suppose you want to mine Twitter for the types of presents people received for Christmas:

> What people on Twitter wanted for Christmas, and what they got: [twitter.com/edchedch/statu…](http://t.co/EGeKTBgF)
>
> — Edwin Chen (@edchedch)
>
>  
>
> January 2, 2012

(Yes, I just embedded a tweet. BOOM.)

How can you figure out which words refer to gifts?

To gather data for the graphs above, I simply looked for phrases of the form “I want XXX for Christmas” and “I got XXX for Christmas”. However, a more sophisticated CRF variant could use a GIFT part-of-speech-like tag (even adding other tags like GIFT-GIVER and GIFT-RECEIVER, to get even more information on who got what from whom) and treat this like a POS tagging problem. Features could be based around things like “this word is a GIFT if the previous word was a GIFT-RECEIVER and the word before that was ‘gave’” or “this word is a GIFT if the next two words are ‘for Christmas’”.

# Fin

I’ll end with some more random thoughts:

- I explicitly skipped over the graphical models framework that conditional random fields sit in, because I don’t think they add much to an initial understanding of CRFs. But if you’re interested in learning more, Daphne Koller is teaching a free, online course on [graphical models](http://www.pgm-class.org/) starting in January.
- Or, if you’re more interested in the many NLP applications of CRFs (like part-of-speech tagging or [named entity extraction](http://en.wikipedia.org/wiki/Named-entity_recognition)), Manning and Jurafsky are teaching an [NLP class](http://www.nlp-class.org/) in the same spirit.
- I also glossed a bit over the analogy between CRFs:HMMs and Logistic Regression:Naive Bayes. This image (from [Sutton and McCallum’s introduction to conditional random fields](http://arxiv.org/pdf/1011.4088v1)) sums it up, and shows the graphical model nature of CRFs as well.