---
layout: post
title:  "My improvised study of syntax analysis (part 2)"
---

![Teddy](/files/2015-04/teddy.jpg)

Parser API
----------

I continue [my previous post](/2015/03/28/syntax-analysis.html). Now it's time to focus on technical details of the implementation of my parser. I decided to use the Go language because I wanted a fast concurrent (i.e. able to utilize all available CPU cores) solution without complicating with C++. Here is my parser API:

{% highlight go %}
type Attribute struct {
	Name   string
	Value  string
}

type ParseMatch struct {
	Text            string
	Nonterminal     string
	Rule            string
	Attributes      []Attribute
	Submatches      []ParseMatch
	Hypotheses      []string
	HypothesisCount uint
}

func Parse(text, nonterminal string, hypotheses_limit uint) []ParseMatch
{% endhighlight %}

*Match* object aggregates a number of sub-matches which correspond to non-terminals and lexical terminals in a matched rule. In the general case, text can have multiple parsings (because of ambiguity of natural languages, e.g. presence of homonyms, etc.). That's why the parsing function returns multiple matches. The above ambiguity should be eliminated at a subsequent semantic phase of text analysis.

So the *Parse* function takes a text to be parsed, a name of non-terminal which needs to be matched (e.g. "sentence") and a maximal number of hypotheses the parser can offer. The *nonterminal* parameter can be empty. In that case the *Parse* tries to match the text to an existing lexical terminal taken from morphology base.

In our terms *hypothesis* is an assumption that a given attribute constraint is satisfied in spite of a real noncompliance (which is considered to be casual). So if the parser encounters a failure of some attribute constraint when the given *hypotheses_limit* is not yet reached it ignores this noncompliance and just continues matching the current rule. In practice this mechanism is pretty convenient for grammar debugging, but drastically reduces parsing performance, so it should be avoided in a real work.

Matching non-terminals
----------------------
Before deepening into the process description it's important to mention that in general case only a beginning part of a text is being matched (not the whole text). The parser just finds all matching prefixes of a given text, which correspond to a given non-terminal.

For a given non-terminal the parser finds all corresponding rules and tries to match them to a given text one after another. Then for each non-terminal's rule the parser reads attribute value assignments ("@{...}", must be at rule's beginning) if they are present. Afterwards it calls a recursive function named *parseRulePart*, which performs the following steps:

1. It matches immediately going non-lexical terminal if it's present.
2. It matches immediately going non-terminal or lexical terminal by calling the *Parse* function.
3. For each match returned by the *Parse* it checks compliance to rule's attribute value constraints.
4. For each compliant match it calls itself in a new goroutine to continue parsing the rule.
5. It returns all matches collected from the launched goroutines merged with the current match.

<span></span>

Matching lexical terminals
--------------------------
On the one hand, lexical terminal can be easily extracted from text. To get a right boundary it's simply needed to find a first occurrence of word separator like space, semicolon, etc. On the other hand, there are idioms and set expressions, which contain such separators inside. They can differ from their separated component words by attribute values and meaning (the second is especially important for subsequent semantic analysis). That's why they have to be considered as consolidated lexical terminals. The matching process consists of the following steps:

1. Extract terminal using word separators as a boundary.
2. Find all morphology forms, which begin with the extracted word.
3. Select all forms matching the beginning of the given text.

<span></span>

Morphology base
---------------

I use the Russian morphology dictionary taken from [OpenCorpora](http://opencorpora.org/). These data had to be minified and indexed for a fast lookup. At first I tried to use SQLite for that purpose, but this solution showed unacceptable performance (even after enabling cache). Then I wrote an add-hoc morphology base, which is about 9 times faster.

The morphology base format is pretty straight-forward. The base file contains a header plus two big parts: text (word-forms, separated by '\0' character) and sorted array of entries (32-bit word-form offset inside the text + 32-bit struct of attribute values). Both parts are loaded into RAM to achieve high lookup performance.

The developed base exposes the following API:

{% highlight go %}
func BuildMorph(txt_filename, morph_filename string) error
func InitMorph(morph_filename string) error
func FinalizeMorph()
func FindTerminals(prefix, separator string) []ParseMatch
{% endhighlight %}

<span></span>

The *FindTerminals* takes an additional separator because in case of multi-word lexical terminal it appears to be a part of it, while otherwise it should be thrown away.

Parsing cache
-------------

Even after switching to ad-hoc morphology base the performance was still unacceptable (it took about several seconds to parse extensive sentences). The nature of the developed grammar implied repeated parsing of the same non-terminals (non-terminals often have many similar rules), so I needed a cache to store temporal results. Thus before processing the *Parse* function checks if the given input exists in the cache.

The cache is based on hash table and to support concurrency it has to be guarded by mutex. But such a global lock can become a bottle-neck. That's why the cache consists of multiple *banks*: each *bank* has its own hash table and a guarding mutex.

The parser usage
----------------

To use the developed [parser](https://github.com/ababo/idiot) you need to:

1. Download the [morphology dictionary](http://ababo.github.io/files/2015-03/dict.opcorpora.txt.zip).
2. Build a morphology base (morph.bin, see *main.go*).
3. Modify the grammar in *rules.yaml*.
4. Stay curious and have fun.