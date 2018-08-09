# PII

Personally Identifiable Information (PII) is a term that we enterprise
developers are hearing about more and more these days. It's a legal term that
refers to a set of attributes that can be used to personally identify a person.
The definition of what exactly constitutes a PII somewhat varies from
jurisdiction to jurisdiction but it commonly includes such things as full name,
email, address, phone number, national id, date of birth.

Why is PII important for developers? Because of various national laws that
prescribe us how to handle it: what and when can be requested, how it can be
stored, processed, archived and destroyed. The main goal, of course, is to
reduce risks, associated with the loss of privacy and identity theft. There are
numerous policies, protocols, and technologies that deal with how to properly
store PII. But here I want to focus on one, often overlooked, aspect of PII
handling. Namely, how NOT to store it.

The problem with information in enterprise applications is that it's like a
water. It's everywhere: in databases, key-value stores, flat files,
logs etc. And, like a water, it always finds a way to leak to places where it
shouldn't be. You can have all the discipline in the world, but one day one
you'll end up with 120Gb of logs contaminated with social security numbers. And
you'll be lucky if you notice it when you are checking your logs, not the
news.google.com.

Bottom line is, your PII will leak. You can contain it, but it will leak
anyway. We need to constantly be on a lookout. Obviously, eyeballing is
not a scalable strategy. We need better ideas. One such idea I want to
explore today is an unstructured data PII detector.


## The problem

We have text data of unknown/varying structure and we need to identify
pieces of PII and deal with them. Dealing is easy part. It could be
raising errors, reporting, or masking.

It's the identification part that is hard. Let's see how we can try to
tackle it.


## The plan

How about we treat this text as a language with a rather bizarre grammar?
We don't know its structure ahead of time. It could be Apache style log,
it could be Splunk key-value pairs, it could be natural language text, csv,
anything really. What we do know is that it may have fragments that _look_ like
PII and we think we know how PII looks like.

Let's try to build a PII lexer. Just a quick refresher: lexer is a tool
that does _lexical analysis_. It consumes a raw stream of characters and
converts it to a list of _lexemes_ (also known as _tokens_). And lexeme is
a sequence of characters with assigned name. In natural languages we would
assign names like _noun, verb, adverb etc._ In computer languages we would
use _keyword, literal, comment etc._ In our world we would use names like
_fullname, emailaddress, dateofbirth etc._


So what makes our grammar bizarre? Two things.

First, it's non-deterministic. We can only _probabilistically_ parse it.
Is _98-12-09_ a date of birth, a phone number or a SKU? Is _Private_
a last name or a part of a sentence that describes the differences between
private and public property? This can be resolved by various rules like,
erring on the side of caution, using overrides etc.

Secondly, it's incomplete. Our lexical analysis will result in a string of
recognized lexemes in a sea of unrecognized characters. This is not
a problem at all, but rather an observation.

With that in mind let's proceed to...


## The implementation

### The Scanner

Let's start building a trivial regular expression based lexer which
essentially loops over a `token: <regex>` hash and labels all substrings
that match this regex as a `token`.

Here is a sketch (Ruby):

```ruby

# text badly contaminated with PII
text = 'Jon Doe email is jon@example.com and his phone is 556-321-9876'

# our PII lexicon
lexicon = {
  email: /\w+@(?:\w+\.)+\w+/,
  phone: /\d{3}-\d{3}-\d{4}/
}

result = {}

# a parser
lexicon.each do |lexeme, regex|
  result[lexeme] = text.scan(regex)
end

result
#=> {:email=>["jon@example.com"], :phone=>["556-321-9876"]}
```

Not bad. We found both email and phone. Obviously, in real life those regexes
ought to be a lot more complex to account for endless national and regional
formats. However, there is a bigger problem than simply inadequate regexes.
Notice how I conveniently ommited perhaps _the_ most important piece of PII - a
full name. As you probably guessed a regex that matches all names (for whatever
reasonable definition of 'all') is not going to be practical.

Obviously, our naive regex lexer is not good enough to deal with proper names.
Time to upgrade it to...


### The Evaluator

So, we can't tell names. Is there anything we _can_ tell? Well, at least we know
that names are _words_ separated by spaces. Let's use that insight:

```ruby
# add new lexeme to our lexicon
lexicon.merge!(name: /[a-zA-Z]+/)

# run it
#=>
{:email=>["jon@example.com"],
 :phone=>["556-321-9876"],
  :name=>["Jon", "Doe", "email", "is", "jon", "example", "com", "and", "his",
  "phone", "is"]}
```

Oh, good. We got his name now. But we also scooped a whole bunch of non-names.
The only option we have at this point is dictionary search to confirm that the
word is the first or last name indeed. This is where things get a little tricky.
We'd need to collect as many names from various sources as possible and search
them for any candidate words have.

How do we efficiently determine if a given string is present in a potentially
giant dictionary without blowing up our little script?
We could use [Trie](https://en.wikipedia.org/wiki/Trie) which seem uniquely
suited for this kind of task. First, let's

```bash
> gem install rambling-trie
```

this is a Ruby gem implementing trie. Next, we need to ingest names to it:

```ruby
require 'rambling-trie'

Rambling::Trie.dump(Rambling::Trie.create('names.txt'), 'names.dump')
```

This converts our `names.txt`, a text file with one name per-line, to a more
efficient marshalled dump of trie. Next, we can just filter out all non-names:

```ruby
trie = Rambling::Trie.load('names.dump')

result[:name].select! { |name| trie.include?(name) }

#=> {:email=>["jon@example.com"], :phone=>["556-321-9876"], :name=>["Jon", "Doe"]}
```

Great! We've identified all pieces of PII in our little example and are now in
a position to do something about them. Let's say we want to mask them:

```ruby
result.values.flatten.each do |pii|
  text.gsub!(pii, '*' * pii.size)
end

=> "*** *** email is *************** and his phone is ************"
```

## Lessons learned so far

Here is what we learned from these exercises.

### Scanning lexers are not enough

Regular expressions alone are not enough to find all lexemes. That's why we are
using them to quickly _scan_ the input and grab everything that looks
like a candidate. Then we further _evaluate_ our candidates to filter our false
positives using plain Ruby.

Examples of lexemes that can probably be adequately identified using regular
expressions alone include phone numbers and email addresses. Although phone
numbers' regular expressions can get (quite
hairy)[https://en.wikipedia.org/wiki/National_conventions_for_writing_telephone_numbers].
Also, one can imagine overly sophisticated evaluator that checks DNS or yellow book records.


### Lexing in general is not enough

In order to improve ambiguous lexemes resolution accuracy we may need to do
some _parsing_. For example, word 'Private' could be a regular word or it could
be last name. However, in order to decide one way or another we may need to
look around. If we see 'Private <first-name>' yeah, it's a last name, all
right. However, if we see 'Private property' then nope.

### It doesn't matter

We don't need a 100% error proof solution. Remember, this is not an adversarial
situation. Nobody is going to try to avoid detection by deliverately inserting
funny delimiters and doing ROT13 substitution. If we accidentally leak PII to
our logs we are likely going to do it big time and even our primitive filter
will find it out quickly without having to resort to fine semantic nuances.


ps: I figured I'll put my (code)[https://github.com/smazhara/stockade] where my
mouth is.
