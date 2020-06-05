# Complexity

In the 2nd Century AD, Ptolemy created a model of the solar system that survived until the 16th century. In this model Ptolemy stated that the Earth was at the center of the universe and that the planets and the Sun orbited around the Earth in perfect circles. In order to make observations match the model, the concept of "epi-cycles" was invented, whereby you could have orbits within orbits (within orbits, etc). This model did a fair job of approximating the motion of the planets, but was heinously complex in comparison to the much simpler model: the Sun is in the center of the solar system and the planets have elliptical orbits, not circular ones. Is the answer to your problem to use fewer moving parts and shift your thinking, or to keep doubling down on complexity until the system submits?

Hypothesis: complexity is the most important property you should be considering when designing systems. By choosing tools and methods that reduce complexity you're setting yourself up for future success at the expense of up front work on design and learning. Once you introduce complexity it's very hard to reduce it again.

The aim of this article is to reason through the thinking that lead to the above hypothesis and make an attempt at putting the notions in there on an objective footing. I hope this allows teams that build systems to have rational debates about how the choices they make will impact complexity, and to care about complexity.

## Setting the stage

A discussion on complexity is predicated on a shared vocabulary and understanding of the ideas used. Let's start by making sure we're talking about the same thing.

Definitions:
- **Complexity**: A measurable property of a system that describes how "entangled" that system is. Notably it's an objective property, not a subjective one. Measured on a scale of _simple_ to _complex_.
- **Difficulty**: A subjective property of a person's ability to achieve a task. Measured on a scale of _easy_ to _difficult_. Your easy might be my difficult. Perhaps you play guitar and find it easy, I find it difficult.
- **Inherent complexity**: The minimum amount of complexity you _must_ contend with in order to to solve the problem for which the system was designed.
- **Incidental complexity**: Complexity that you've added to a system on top of the complexity required to solve the problem for which the system was designed.
- **Orthogonal**: "At right angles to". When two things are orthogonal, a change in one thing says nothing about a change in the other. If you increase the _mass_ of a lump of iron, you say nothing about its _density_, the two concepts are "orthogonal".

### Complexity isn't cardinality - complexity isn't counting - more is not more complex (necessarily)

One of the logical mistakes people make when attempting to reason about complexity is thinking that more things _is_ more complex. Let's try and demonstrate why that isn't (necessarily) true:

In our toy system, we have strings hanging down from a bar. In order to gain value from the system, you have to match the number at the bottom with the number at the top.

System 1
```
1  2  3  4  5  6 ... n
----------------------
|  |  |  |  |  |     |
|  |  |  |  |  |     |
|  |  |  |  |  |     |
|  |  |  |  |  |     |
|  |  |  |  |  |     |
|  |  |  |  |  |     |
|  |  |  |  |  |     |
1  2  3  4  5  6     n
```

Imagine we used a special tool to build the system that guaranteed the strings could not be swapped over (e.g. a software compiler, a good design, a robot), if I had 100,000,000,000 strings, the answer would still be trivial to determine _without even looking at the system_. If I asked you: "for top number 9,834,654 what's the bottom number?" You would barely have to think to answer "9,834,654". It doesn't matter how many strings I have, the system is no more complex than if there was one string. The complexity of this system is 0.

Now imagine we created a system whereby for each row of text you were allowed to either:
- stay going straight down
- swap with the string to your left
- swap with the string to your right

Here's an example of a possible outcome that's easy enough to draw in ascii-art (but is subtly wrong because it takes two rows to do a swap in this drawing but should only take one - please forgive me and use your imagination):

System 2
```
1  2  3  4  5  6
----------------
|  |  |  |  |  |
 \/   |  |  |  |
 /\   |  |  |  |
|  |  |  |  |  |
|  |  |   \/   |
|  |  |   /\   |
|  |  |  |  |  |
1  2  3  4  5  6
```

Some important properties about this system have changed:
- It's no longer trivial to answer the question "For top number `n`, what's the corresponding bottom number?"
- The number of states the system can be in has gone from 1 to 13,122 (6 * 3^7, might help you to think of the tree of possible options if you care about checking the number. Also assumes that you can to to 0 and 7 on the extreme left/right).
  - Let's reiterate that quickly: six components, 7 opportunities to make a choice, three possible choices and 13,121 more states...yikes! Don't give your systems options!
  - You may or may not wish to treat with suspicion a person who claims they can reason about 13,122 states.

If someone offered you a dollar per question for a correct answer and you got to choose between System 1 with 6 strings and System 2 with 1,000,000,000 strings, which would you choose? Hopefully I've demonstrated that cardinality (the number of things you have) and the complexity (how intertwined they are) are orthogonal concepts.

### Measuring complexity

I made the rather bold claim in the definition above that complexity is a _measurable_ property of a system, and that it's _objective_ not _subjective_. Let's explore that briefly and see if we can give some examples.

#### Method 1 - The number of states of the system

One possibly valid measure of complexity is counting the number of states a system can be in:

Example:
- If I have three squares, and two buttons (x and y), how many states can my system be in?
```
----------
|x_|__|_y|
```

This system can be in 6 states: (`x - y`, `- x y`, `x y -`) and the same again where you swap the x and y.

Example 2:
```
type Input = 'A' | 'B';

user-input <- read_input_from_console()

validate_either_A_or_B(user-input) // error if invalid

function answer(input: Input) {
    internal-state <- random-choice(['A', 'B'])
    return is-equal(internal-state, input)
}

print(answer(user-input))
```

This is a slightly trickier example to parse, and we may have to make some assumptions:
- There are an infinite number of inputs that could have come from the console
- After validation there are only two states that can possibly make it into the answer function (and the error state adds an extra state)
- The system can be in 4 internal states inside the answer function (A/B, B/A, A/A, B/B)
- The outside world can only see three states (true/false/error)


What about if it didn't have the validation?

How could you add to the number of states of the system without changing it's external interface? How could you reduce the number of states?

The most important question: for _the exact same behaviour_ to the user, is it better to have a system that can be in more or fewer states, or doesn't it matter? What are the implications for validation and use of a type system?

#### Method 2 - Coupling between components

Coupled components can be objectively identified because they "know about" each other. Thanks to information theory and work done on compression, knowledge can be measured in bits. If we have two software modules:

```
Module A {
    a-property = 10
}

Module B {
    how-many-widgets() {
        if A.a-property == 10        // <- Module B knows about Module A, they share 1 bit of information 
            'some widgets'           // (1 bit has two states, in this case "is 10", "is not 10")
        else
            'lots of widgets'
    }
}
```

In the example above you can't reason about the behaviour of module B without also knowing about the sate of module A _at the time_. And this is the absolute killer of mutable state, you have to know both the input values to the system _and_ the state of the system _at the time_. If you deal in only immutable values and pure functions, you only need to know the input values and your output is _defined_ (see [referential transparency](https://en.wikipedia.org/wiki/Referential_transparency)).

#### Method 3 - Using both knowledge and number of states

Perhaps there is a higher abstraction that describes both knowledge and the number of states. I currently haven't thought about how to mathematically/rationally combine these two measures, but I suspect it's doable for someone literate in the ideas and who has the time.

#### Summary of measuring complexity

Hopefully I've given the thin end of the wedge for thinking about how you might measure the complexity of a system objectively. It very quickly gets very difficult to put actual numbers against real world systems. However, what's much easier is to ask "if I take action X, will it _increase_ or _decrease_ the complexity?", where X may be: add some code, delete some code, use some database, add a feature, etc.

### How to measure complexity when you combine two systems

The short and sweet answer is: you **multiply together the complexities of the two systems**. It's actually more complex than that, because there may be states that are mutually exclusive when you combine the systems so certain things can't happen.

If you keep multiplying small numbers together they become quite large numbers fairly quickly. What seems like a little bit of complexity when only thinking about one module can make a system go from a complexity of 100 (just about manageable) to 300 (getting to be too hard for a human being) by only adding a complexity of 3.

Let's take system 1: a function that spits out A or B, and system 2: a function that spits out 1 or 2. Each system has a complexity "2" (1, 2) | (A, B) on it's own, but if you call one then the other and use the result, it has complexity 4 (1A, 1B, 2A, 2B). If you can call them in _either_ order, obviously the complexity goes up even more.

This is a recursive definition. If you need to combine 3 systems, simply combine any two of them, and then combine the combination with the third. You can do this `n` times for `n` systems.

A complex system can be measured by the number of possible states it can be in. Therefore, if you take more complex things and put them together, you will have a more complex system than if you take simple things and put them together.


## More ideas, concepts and materials to enrich your thinking around complexity

### Concepts
- entropy
- information theory
- dependencies
- how time, concurrency and mutable state interact

### Questions
- What do type systems do to address complexity? What about with un-validated inputs?
- What do tests do to address complexity?

### Materials

- [Simple made easy](https://www.infoq.com/presentations/Simple-Made-Easy/) A talk about simplicity in software systems given by the author of programming language Clojure, Rich Hickey. This was the start of my journey thinking about complexity in an objective way. I recommend watching it multiple times!
- [Complexity bias](https://fs.blog/2018/01/complexity-bias/) An article about complexity bias, how people tend to favour more complex systems and explanations over simpler ones.
- [Out of the tar pit](https://github.com/papers-we-love/papers-we-love/blob/master/design/out-of-the-tar-pit.pdf) A paper on how we can build better software systems.

## In conclusion
Hopefully you can now see some merit in the idea that complexity is no more a matter of opinion than `1 + 1 = 2` is a matter of opinion, assuming you set your system of thought up rigorously enough. If you disagree then perhaps you are _con-fused_ literally "thoroughly-mixed" (up) because I haven't found a suitably simple way to explain complexity, or maybe I'm wrong. Either way, I would love to hear about your thoughts. Open a GitHub issue or a PR if you like.

To understand complexity is partly to understand oneself. You have to be ready to address some deeply entrenched thought patterns if you're going to be able to see past complexity to underlying simplicity (see complexity bias above). I hope this document has interested you and given you a basis upon which you can grow your ideas about systems and have as much fun with the ideas as I have.


# Appendix

## A: Quotes

> Simplicity is a great virtue but it requires hard work to achieve it and education to appreciate it. And to make matters worse: complexity sells better.
>
> -- <cite>Edsger W. Dijkstra</cite>

> Out of complexity, find simplicity.
>
> -- <cite>Albert Einstein</cite>

> Any darn fool can make something complex; it takes a genius to make something simple.
>
> -- <cite>Peter Seeger</cite>

> Most geniuses, especially those who lead others, prosper not by deconstructing intricate complexities but by exploiting unrecognized simplicities.
>
> -- <cite>Andy Benoit</cite>

> That's been one of my mantras -- focus and simplicity. Simple can be harder than complex; you have to work hard to get your thinking clean to make it simple.
>
> -- <cite>Steve Jobs</cite>

> Less is more
>
> -- <cite>_old saying_</cite>
