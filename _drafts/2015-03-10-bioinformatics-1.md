---
published: false
---





## Practical Biology for software developers

I have started tinkering with the software at the age of fourteen, and while I wasn't particulary good at it, I was **astonished** by how one can create things out of a thin air. Dynamic web pages were the thing, so I got a book about PHP3 and made a CMS. I was into computer games as well, and when a one well-known MMORPG *(totally not World of Warcraft)* came out, I wanted to play. I wanted to play so bad, but I didn't have the cash, so I started working on an open-source server instead. There was something compelling about a world that almost felt alive, it was a software I **wanted** to do so much, that I've learned C++ because of it. Why?

> The older I get, the more I believe that the only way to become a better programmer is by not programming.
> - "[How to become a better programmer by not programming](http://blog.codinghorror.com/how-to-become-a-better-programmer-by-not-programming/)"

### The Many Facets of Performance

An organism is, depending on who do you ask, a system of autonomous units called *[cells][cell-theory]* which in some way promote reproduction or some other method of survival. The key thing is that they form a *semi-closed* system, that means they're not ignorant of what's going outside and they can interact with each other. Compare this to a function with a state (first-class function), or an object in computer languages. These tiny objects can *differentiate* (and *dedifferentiate*) depending on the environment and signalization mechanisms. This is a fancy way to say they can be refit for various tasks, just like the objects in prototype-based languages, such as JavaScript or Lua.
This is probably the **most** important pattern in organisms, even on the macroscopic level, responsible for the resiliency and aptitude to survive. It shows how the organisms deal with failures - a malfunctioning cell triggers a "[programmed cell death][apoptosis]", a cell takes one for the team.

> In an increasingly multi-core world, the ability to isolate the processes as well as shield each open tab from other misbehaving pages alone proves that Chrome has a significant performance edge over the competition. In fact, it is important to note that most other browsers have followed suit, or are in the process of migrating to similar architecture.
> - [High Performance Networking in Chrome, The Many Facets of Performance](http://www.aosabook.org/en/posa/high-performance-networking-in-chrome.html)

Google Chrome design copied copied this pattern in time of the threaded browsers, and it paid off. In fact it goes even farther with *[zygote][zygote-chrome]* process.
A *zygote* is a cell containing a complete genetic information, formed from two reproduction cells called *gametes*. Similarly, the zygote process opens all files and performs some initialization, and then it can be `fork()`-ed as needed, with all the information already in place.

### Disassembling our code

Producing "code" is a discipline that requires critical thinking combined with language skills and plain old engineering. The language part sets it apart from other engineering disciplines like construction or electric engineering. We often say: "I'm a C guy" or "He's a Clojure ninja", but why is the dialect we speak on the front page more often than what we can do with the language? Computer languages, just like natural languages, differ in expressiveness, number of speakers, idioms and trends.
But all the languages have one thing in common - they are always translated to a language that computers can understand, a machine code. As this is essentially a one-way process, the computer languages do not have to encode colour, emotions nor secondary meanings. They're confusing for us because the semantics is for computer, byt syntax for humans.

Since the advent of the DNA sequencing and central dogma, we've been curious what's *our* code. The [Human Genome Project][hgp] was declared complete more than a decade ago now, but it's just the first step. A genetic information is included in our cells containing a *nucleus*, a core. It's very tiny of course and it's [packed very tightly to fit][dna-packaging], which makes it impractical to read.
Fortunately it also encodes a machinery to unravel it, clone it, fill it and cut it.
Using it enabled all kinds of sequencing methods, notably [shotgun sequencing][shotgun]. Instead of tedious clone-by-clone mapping, it breaks the DNA sequence randomly into many small pieces and reassembles the sequence from overlapping fragments. In another words, it accelerates the process by sheer paralelization, an important pattern often found in organisms.

Quick facts that we know - compared to binary code, genetic information is encoded in a sequence of 4 basic base pairs. This has lower [radix economy][radix-economy] than a binary system. Each triplet of base pairs then encodes an amino-acid, this would have been the most economical choice if it wasn't for the fact that it's redundant. A single AA can be encoded by a multitude of triplets. CPU instructions, on the other hand, are encoded by exactly one unique sequence of bits.
Sequences of AA form peptide chains, proteins and eventually a complexes of proteins.

I'm going to stop here. What we don't have is a "CPU reference manual" to decipher the semantics of the code. For example, we can have a look in the x86 manual and see that the `ADD` instruction performs addition, this makes disassembly (and decompilation to some degree) possible, but have no such knowledge about many motifs found in the genome.





[cell-theory]: http://www.biologyreference.com/Gr-Hi/History-of-Biology-Cell-Theory-and-Cell-Structure.html
[apoptosis]: https://www.khanacademy.org/test-prep/mcat/biomolecules/Krebs-citric-acid-cycle-and-oxidative-phosphorylation/v/mitochondria-apoptosis-oxidative-stress
[zygote-chrome]: http://neugierig.org/software/chromium/notes/2011/08/zygote.html
[dna-packaging]: http://www.nature.com/scitable/topicpage/dna-packaging-nucleosomes-and-chromatin-310
[hgp]: https://en.wikipedia.org/wiki/Human_Genome_Project
[radix-economy]: http://hummusandmagnets.tumblr.com/post/48664858476/the-most-efficient-radix-is-not-e
[shotgun]: http://www.nature.com/scitable/topicpage/complex-genomes-shotgun-sequencing-609