Title: Parallel processing with python
Date: 2014-09-02 16:00
Category: software development
Tags: python parallel

Since i do a lot of machine learning and nlp related tasks, i often need to preprocess large amounts of data. For web pages, which where obtained by a crawler, these steps might include:

* remove html code and extract the raw text data
* [tokenize](http://nlp.stanford.edu/IR-book/html/htmledition/tokenization-1.html) the raw text data
* filter unwanted tokens such as punctuation characters
* [stem](http://en.wikipedia.org/wiki/Stemming) the remaining tokens

The following is a very simple example. Lets assume we have a text file where every line is a document:

    :::text
    This is an example text.
    Another text with no meaning!
    What is this? Some more text here.

The goal is to extract every word and write those tokens separated by whitespace to a new file.

    :::python
    import re

    tokenized_lines = []
    with open('input.txt') as f:
        for line in f:
            tokenized_lines.append([token for token in re.split('\W+', line)
                                    if token.isalpha() and len(token) > 2])
                  
    with open('output.txt', 'w') as f:
        for line in tokenized_lines:
            f.write('%s\n' % ' '.join(line))


It would obviously better not to accumulate all tokens in memory, but to write them to a file as soon as they are extracted. But for simplicity i will keep it this way. With enough data this step will take a long time. To reduce the execution time i tried to run the tasks in parallel, since every modern laptop has multiple cores and the tokenization task is cpu bound. 
    
#### Requirements

Before i started to search for a solution i thought about the requirements it should have.

##### Overhead
I did not want to use one of the many existing [solutions](https://wiki.python.org/moin/ParallelProcessing). I rather searched for a solution based on the standard library with minimal overhead. I also did not need support for clustering, since i was restricted to one machine.

##### Multiprocessing
If you don't know about the GIL, you should read this [blog post](http://www.jeffknupp.com/blog/2012/03/31/pythons-hardest-problem/). Since there can be only one thread active at the time in python programs, the threading library is useless for cpu bound tasks. Instead one should use the [multiprocessing module](https://docs.python.org/2/library/multiprocessing.html).

##### Streams
The data size was always too large to fit into memory. Because of that, i always used data streams, mostly contents of text files. The new solution should not only be able to support streams, but should also give the user the opportunity to regulate how much memory is used.

#### Solution

Based on my requirements i wrote a simple solution which is hosted on [github](https://github.com/raphaelbrand/PyProcessParallel). The architecture of the solution is best described in the following picture.

![task chain Text]({filename}/images/PyProcessParallel.png)

The producer reads from a source and puts work into the work queue. It is possible to specify a size for each queue. If a queue is already at maximum capacity an attempt to put more items into will block the caller. This allows me to control the amount of memory which will be used. Every worker runs in its own process and takes work out of the work queue. After a worker has finished its task, it puts the result into the result queue. The consumer will then collect the results and write them to specified destination.

Lets transform our example from the beginning to use the new script. I'll assume you already downloaded the script from github and included it in your project:

    :::python
    from process_parallel import process_parallel
    
    # you could use the file object as the producer,
    # no need to write a generator.
    # But for demonstration purposes i'll write a function
    # to show you how to include other sources
    input = open('input.txt')
    def producer():
        for line in input:
            yield line

    def worker(line):
        return [token for token in re.split('\W', line)
                if token.isalpha() and len(token) > 2]

    out = open('output.txt', 'w')
    def consumer(tokens):
        out.write('%s\n' % ' '.join(tokens))

    # the producer should support __iter__
    # worker and consumer need to be functions 
    process_parallel(producer(), worker, consumer
                     maxsize_jobs=100, maxsize_results=200)

    #clean up
    input.close()
    out.close()


The solution is simple enough, only 50 lines of python code. The queues allow us to control the memory being used. On the downside there is a overhead due to the use of the processes. The data needs to be serialized and deserialized between processes. This means, if you're worker logic is too simple, then your code will run slower in parallel.
