---
published: false
---



## Practical Biology for software developers, part 1

> <cite>"The older I get, the more I believe that the only way to become a better programmer is by not programming."</cite>
>
> - Jeff Atwood, [How to become a better programmer by not programming](http://blog.codinghorror.com/how-to-become-a-better-programmer-by-not-programming/)

Copying is innovation and innovation is copying. Engineering disciplines often take a cue from traits and patterns found in life.

<p style="text-align:center">
<img src="http://www.bloomberg.com/image/ib9JnQnKe6jE.jpg"/>
<br />
<cite>HIROMI OKANO/CORBIS; WEST JAPAN RAILWAY CO. VIA BLOOMBERG</cite>
</p>

Eiji Nakatsu, an engineer at the JR-West rail company, redesigned the the nose of the train by the beak of the kingfisher to reduce drag and noise. It's an engineering feat that would otherwise take numerous simulations and incremental optimisations. While it's false to think that nature always has the best answers to man-made problems, it simply had more time to figure out what does and doesn't work.
Now you might say: *"okay, a guy really liked birds and got lucky, so what?"*, and you might be right. This series of short articles isn't about clear-cut advice, but rather a collection of observations about what makes the living things tick.

### The Many Facets of Performance

An organism is, depending on who do you ask, a system of autonomous units called *[cells][cell-theory]* which in some way promote reproduction, or other method of survival. The key thing is that they form a *semi-closed* system, that means they're not ignorant of what's going outside and they can interact with each other, but they know their boundaries. Compare this to a function with a state (first-class function), or an object in computer languages. These tiny objects[^stem-cells] can *differentiate* (and *dedifferentiate*) depending on the environment and signalization. This is a fancy way to say they can be refit for various tasks, just like the objects in prototype-based languages, such as JavaScript or Lua.
This is probably the **most** important pattern found even on the macroscopic level, and it gives them resiliency, and aptitude to survive. It shows how the organisms deal with failures and end-of-life states - a malfunctioning cell triggers a "[programmed cell death][apoptosis]", and takes one for the team. The real MVP. Even though there are [many safeguards][dna-repair] in cell replication, a failure is not treated as something impossible, but as a normal part of life.

> <cite>"In an increasingly multi-core world, the ability to isolate the processes as well as shield each open tab from other misbehaving pages alone proves that Chrome has a significant performance edge over the competition. In fact, it is important to note that most other browsers have followed suit, or are in the process of migrating to similar architecture."</cite>
>
> - Ilya Grigorik, [High Performance Networking in Chrome, The Many Facets of Performance](http://www.aosabook.org/en/posa/high-performance-networking-in-chrome.html)

Google Chrome design copied copied this pattern in time of the threaded[^threads]  browsers, and it paid off. In fact it goes even farther with the *[zygote][zygote-chrome]* process.
A *zygote* is a cell containing a complete genetic information, formed from two reproduction cells. Similarly, the "zygote process" opens all files and performs some initialization, and then it can be `fork()`-ed as needed, with all the information already in place.

### Disassembling our code

Producing "code" is a discipline that requires critical thinking combined with language skills and plain old engineering. The language part sets it apart from other engineering disciplines like construction or electric engineering. We often say: *"I'm a C guy"* or *"He's a Clojure ninja"*, but why is the dialect we speak so important at all? Computer languages, just like natural languages, differ in expressiveness, number of speakers, idioms and trends.

> <cite>"The big picture is lost in the process"</cite>

But all the languages have one thing in common - they are, in the end, translated to a language that computers can understand, a machine code. As this is essentially a one-way process, the computer languages do not have to encode colour, emotions nor secondary meanings. They're confusing for us because their semantics is for computers, byt syntax for humans.

Since the advent of the DNA sequencing and central dogma, we've been curious what's *our* machine code. The [Human Genome Project][hgp] was declared complete more than a decade ago now, but it's just the first step. A genetic information is included in our cells containing a *nucleus*, a core. It's very tiny of course, and also [packed very tightly to fit][dna-packaging], which makes it impractical to read. Fortunately it also encodes a machinery to unravel it, clone it, fill it and cut it. A toolkit of some sort.
Using it enabled all kinds of sequencing methods, notably [shotgun sequencing][shotgun]. Instead of tedious clone-by-clone mapping[^mapping], it breaks the DNA sequence randomly into many small pieces and reassembles the sequence from overlapping fragments. In another words, it accelerates the process by sheer paralelization, another ubiquitous pattern.

Quick facts that we know - compared to binary code, genetic information is encoded in a sequence of 4 basic base pairs. Each triplet of base pairs then encodes an amino-acid (AA), this would have been the [most economical choice][radix-economy] if it wasn't for the fact that it's redundant. A single AA can be encoded by a multitude of triplets. CPU instructions, on the other hand, are encoded by exactly **one** unique sequence of bits. Nature isn't irked **at all** about storage efficiency. Now what we don't have is a "CPU reference manual" to decipher the semantics of the code. For example, we can have a look in the x86 manual and see that the `ADD` instruction performs addition, this makes disassembly (and decompilation to some degree) possible, but we have no such knowledge about many motifs found in the genome. In order to even recognize them, we need to learn how to identify patterns first.

### The waggle dance

Swarming animals leverage simple patterns for communication, and provide a good model in the age of anycast network routing, containers, clusters and IoT. The problem of foraging bees is glaringly similar to a man-made problem of cost and routing in the computer networks. The foragers communicate not only the distance and orientation of the food source, but also its quality through a specific dance. This motivates other foragers to switch to the current best food source, and dance in return to motivate even more workers.

<p style="text-align:center">
<iframe width="560" height="315" src="https://www.youtube.com/embed/2S-ozxpIrdI" frameborder="0" allowfullscreen></iframe>
</p>

The [ZigBee protocol][zigbee] borrowed not only it's name from the apis, but also the behavior of the repeater radios in an electronic Zigbee network. Granted, it hasn't been overly successful yet.

On the macroscopic level, a behaviour of the hive is useful for making *rule of thumb* decisions. A bread and butter for programmers. When the [hive is foraging][foraging], it has two choices - either stay with the current food source or scout for a better one, which may or may not pay off. The waggle dance of the returning forager communicates the direction to the food source, and an integrated travel length (cost). Based on the net gain, it might recruit more foragers based on several other constraints.

* Foragers stay true to their food source of choice until it worsens its quality.
* More than 3/4 of the foragers prefer to stay in their comfortable distance, and only few experienced ones fly farther.
* The amount of foraging activity is limited by environment.

The same choice paralysis problem exists in computer networking software, where instead of food we're chosing servers for the next hop. The question is always - did we pick the fastest server, and what if something changed meanwhile? What if our choice fails? A good rule of thumb is to stick with the best performing server (in the range) with at least 3/4 probability *(fidelity)*. If not, try different server with the probability proportional to it's known quality *(scouting)*. Back off if all servers are consistently bad *(weather)*.

This simple *rule of thumb* algorithm shows both fast response time to changes and congestion control, compared to traditional approaches like [Exponential decay][exponential-decay], where the round-trip time (cost) of the inactive server decays over time, so the application is incented to revisit it at some point.

## Paterns revisited

I feel that programmers, well... at least me, are sometimes too burdened with the need to make everything right on the first try, and yet it rarely ever works. If it did, there would be no need for versioning or patches. We're ashamed when the coding standard is not good enough, about the `printf("swearword")` debugging strategy, copypasta code. It hurts our pride. Yet even an imperfect model can reveal a lot of important clues about how to make the *final version* suck less. That's why the builders build models and scientists experiment. Looking at the successful patterns in life can help us understand why approaches live or fail before we even start building the model.

[^mapping]: Figuring out where to place the sequence fragment on the chromosome.
[^stem-cells]: These cells are called "stem cells", they're special because they have the potential to become *anything*. You might have heard about these because of the stem cell therapy, a transplantation of these cells.
[^threads]: Threads are finnicky to work with and overused, but more about that later.
[dna-repair]: http://www.ncbi.nlm.nih.gov/books/NBK26879/
[cell-theory]: http://www.biologyreference.com/Gr-Hi/History-of-Biology-Cell-Theory-and-Cell-Structure.html
[apoptosis]: https://www.khanacademy.org/test-prep/mcat/biomolecules/Krebs-citric-acid-cycle-and-oxidative-phosphorylation/v/mitochondria-apoptosis-oxidative-stress
[zygote-chrome]: http://neugierig.org/software/chromium/notes/2011/08/zygote.html
[dna-packaging]: http://www.nature.com/scitable/topicpage/dna-packaging-nucleosomes-and-chromatin-310
[hgp]: https://en.wikipedia.org/wiki/Human_Genome_Project
[radix-economy]: http://hummusandmagnets.tumblr.com/post/48664858476/the-most-efficient-radix-is-not-e
[shotgun]: http://www.nature.com/scitable/topicpage/complex-genomes-shotgun-sequencing-609
[zigbee]: http://electronicdesign.com/communications/what-s-difference-between-zigbee-and-z-wave#1
[foraging]: http://www.agf.gov.bc.ca/apiculture/factsheets/111_foraging.htm
[exponential-decay]: http://mathworld.wolfram.com/ExponentialDecay.html