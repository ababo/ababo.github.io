---
layout: post
title:  "My improvised study of syntax analysis (part 1)"
---

![Artificial Intelligence](/files/2015-03/ai.jpg)

Intro
-----

Always keeping my interest in AI I many times thought about ways of understanding text in natural language. It's well-known that the classical theory of text analysis divides this process into three phases:

* Morphological - word forms and their attributes (e.g. case or tense);
* Syntax (parsing) - sentence structure (inter-word relations);
* Semantic - meaning based on world knowledge (models of reality);

The first phase is mostly solved today. We have both morphological dictionaries which cover the lion's share of word forms we find in any text and set of rules to analyze unknown words.

The situation with the second phase is much more complicated. The existing parsers don't even pretend to be accurate in complex cases. Moreover most of them are proprietary and close-sourced (here I'm focusing on Russian language supposing that the situation with English parsers is slightly better). So in order to advance in the direction of machine understanding of text written in natural language we need to develop good parsers.

Because of my lack of deep understanding of neural net techniques I decided yet to follow more traditional path: to develop BNF-like grammar description and to implement a parser which consumes set of such rules. From this perspective the greater part of the work will consist of composing this set to cover the most of the language (and it's still far from being finished). In the second part of this post I'll discuss the implementation details of my [parser](http://github.com/ababo/idiot), but now I would like to describe the developed grammar notation.

I decided to base my work on concrete examples. As mentioned I wanted to focus on Russian language which is my mother-tongue. So I took "Ward №6" - the famous novel written by Anton Chekhov in 1892 and started to manually parse sentences one by one finding corresponding syntax rules. As it turned out I immediately confronted troubles because of complexity of Chekhov's sentences.

The basics of developed grammar
-------------------------------

Before starting to deepen into the first sentence I should warn the reader that I'm not a professional linguist, so my parsing can be wrong from academic point of view. Here I try to achieve sentences structuring which should be suitable for future semantic analysis rather than to find a parsing which can be accepted by a scholar.

> В больничном дворе стоит небольшой флигель, окруженный целым лесом репейника, крапивы и дикой конопли.

This is a simple sentence which has the subject (флигель), the attribute (небольшой) and the participial phrase (окруженный целым лесом репейника, крапивы и дикой конопли); the second and the third are related to the first. Additionally we have the predicate (стоит) and the related adverbial modifier of place (в больничном дворе).

Let's divide the sentence into two big parts:

* the first one will call "predicate_group" - it will contain the predicate itself and the related adverbial modifier.
* the second will call "subject_group" - it will contain the subject itself and the related attribute and participial phrase.

We can write our first grammar rules already:

```
sentence:
- "{predicate_group} {subject_group}."

predicate_group:
- "{adverbial_modifier} {predicate}"

subject_group:
- "{attribute} {subject}, {participial_phrase}"
```

Let's explain the written above. The grammar is given in YAML syntax. This is a convenient format for human to work with (it's also used by my current [parser implementation](http://github.com/ababo/idiot)). Words which end with colon designate *non-terminals*; dash-prefixed quoted lines represent *rules* corresponded to the non-terminal above. Each rule is a sequence of elements of two types: *terminal* or *non-terminal*. For example the first rule represent the following sequence: non-terminal "predicate\_group", terminal " ", non-terminal "subject\_group" and terminal ".". In general case non-terminal can have several rules. For example, sentence can also start with a subject\_group followed by a predicate\_group.

The current grammar does not take into account many language constraints. For example, the following illegal sentence can be successfully parsed with it:

> В больничном дворе стояла небольшие флигелю, окруженный целым лесом репейника, крапивы и дикой конопли.

So the parts of the sentence should correspond one to another. The correspondence above is related to attributes called *number*, *gender* and *case*. Let's rewrite our grammar taking into account these constraints.

```
sentence:
- "{predicate_group number=@1 gender=@2} {subject_group number=@1 gender=@2}."

predicate_group:
- "{adverbial_modifier} {predicate !number !gender !tense}"

subject_group:
- "{attribute} {subject case=nomn !number=@1 !gender=@2}, {participial_phrase number=@1 gender=@2}"
```

Let's explain the added syntax. As we can see some non-terminals have attributes. Attribute value can be immediate (a concrete value like "nominative case").

Some attribute names start with an exclamation mark. This prefix means "import this attribute from the corresponded non-terminal to the outer one". For example, "predicate\_group" should export number, gender and tense attributes, but they cannot be directly defined (as immediate values) in its rules. But these attributes are defined in "predicate", so we should import them from this child non-terminal.

Sometimes it's important to constrain two or more child non-terminals to have a same attribute value even if it's undefined on this level. For this purpose let's use an at sign prefix. For example, in "sentence" both "predicate\_group" and "subject\_group" should have same values of case, number and gender attributes. Constrained attribute can also be imported (in this case its name starts with an exclamation mark).

Non-strict attribute constraining
---------------------------------

In Russian the verb form depends on the subject's gender being in the past tense (кот ел, кошка ела), but it does not depend on it being in the present or the future tenses (кот ест, кошка ест). We could describe this law in our grammar by separating tenses but it's simpler to use non-strict attribute constraining: attribute constraint is ignored if the attribute is not defined in one of the corresponding non-terminals.

Let's give an example to clarify things. In order to parse phrases "кот ел", "кошка ела", "кот ест", "кошка ест" we'll take the following rule:

```
- "{noun case=nomn number=@1 gender=@2} {verb number=@1 gender=@2}"
```

In case of the present tense the appropriate verb form (ест) will not have "gender" attribute, so because of non-strict constraining the gender constraint will just be discarded. This trick also helps to handle the number constraint: we don't need to add additional rule for plurals (the gender attribute will be discarded because of non-strict attribute constraining). This approach considerably helps to reduce grammar size and complexity.

Attribute value assignment
--------------------------

Sometimes we need to assign values to attributes latter used in our constraints. For example:

```
- "@{number=plur}{noun} и {noun}"
```

This rule states that the number attribute of a phrase which consists of enumeration of two nouns (слон и моська) is plural. Attribute value assignment always starts from rule's very beginning, it has an at sign prefix and it consists of attributes with corresponded immediate values.

Lexical terminals
-----------------

We could define lowest-level non-terminals through terminals like that:

```
predicate:
...
- "@{number=sing gender=femin tense=past}ела"
- "@{number=sing gender=masc tense=past}ел"
- "@{number=plur tense=past}ели"
...
```

But this approach is impractical due to a huge number of different word forms. Such terminals should be taken from a morphological dictionary. That's why these terminals are not listed directly but defined by set of attribute values:

```
predicate:
- "{pos=verb !number !gender !tense}"
```
As we can see this notation is pretty similar to attribute assignment but without an at sign character and without constraint to be at rule's very beginning. The rule above corresponds to terminals with "verb" value of attribute "pos" (part of speech).