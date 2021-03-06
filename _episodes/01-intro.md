---
title: "Introduction"
teaching: 20
exercises: 0
questions:
- "How can I make my results easier to reproduce?"
objectives:
- "Explain what Make is for."
- "Explain why Make differs from shell scripts."
- "Name other popular build tools."
keypoints:
- "Make allows us to specify what depends on what and how to update things that are out of date."
---

Let's imagine that we're interested in
testing Zipf's Law in some of our favorite books.

> ## Zipf's Law
>
> The most frequently-occurring word occurs approximately twice as
> often as the second most frequent word. This is [Zipf's Law][zipfs-law].
{: .callout}

We've compiled our raw data, the books we want to analyze
(check out e.g. `head books/isles.txt`)
and have prepared several C++ and Python programs that together make up our
analysis pipeline.

Our directory has the C++ and Python programs and data files we
we will be working with:

~~~
|- books
|  |- abyss.txt
|  |- isles.txt
|  |- last.txt
|  |- LICENSE_TEXTS.md
|  |- sierra.txt
|- main.cpp
|- plotcount.py
|- wordcount.cpp
|- wordcount.h
|- zipf_test.py
~~~
{: .output}

The first step is to count the frequency of each word in a book. In order to do this we need to compile
the wordcount.cpp program. Assuming your C++ compiler is invoked using the `c++` command, you would do
something like this:

~~~
c++ --std=c++11 -o wordcount wordcount.cpp main.cpp
~~~
{: .bash}

This should result in an executable program called `wordcount`. This program can now be run as follows:

~~~
$ ./wordcount books/isles.txt > isles.dat
~~~
{: .bash}

Let's take a quick peek at the result.

~~~
$ head -5 isles.dat
~~~
{: .bash}

This shows us the top 5 lines in the output file:

~~~
the 3822 6.7579
of 2460 4.34967
and 1721 3.043
to 1479 2.61511
a 1307 2.31098
~~~
{: .output}

We can see that the file consists of one row per word.
Each row shows the word itself, the number of occurrences of that
word, and the number of occurrences as a percentage of the total
number of words in the text file.

We can do the same thing for a different book:

~~~
$ ./wordcount books/abyss.txt > abyss.dat
$ head -5 abyss.dat
~~~
{: .bash}

~~~
the 4021 6.46546
and 2792 4.48932
of 1899 3.05345
a 1555 2.50032
to 1495 2.40385
~~~
{: .output}

Let's visualize the results.
The script `plotcount.py` reads in a data file and plots the 10 most
frequently occurring words as a text-based bar plot:

~~~
$ python plotcount.py isles.dat ascii
~~~
{: .bash}

~~~
the   ########################################################################
of    ##############################################
and   ################################
to    ############################
a     #########################
in    ###################
is    #################
that  ############
by    ###########
it    ###########
~~~
{: .output}

`plotcount.py` can also show the plot graphically:

~~~
$ python plotcount.py isles.dat show
~~~
{: .bash}

Close the window to exit the plot.

`plotcount.py` can also create the plot as an image file (e.g. a PNG file):

~~~
$ python plotcount.py isles.dat isles.png
~~~
{: .bash}

Finally, let's test Zipf's law for these books:

~~~
$ python zipf_test.py abyss.dat isles.dat
~~~
{: .bash}

~~~
Book	First	Second	Ratio
abyss	4021	2792	1.44
isles	3822	2460	1.55
~~~
{: .output}

So we're not too far off from Zipf's law.

Together these scripts implement a common workflow:

1. Read a data file.
2. Perform an analysis on this data file.
3. Write the analysis results to a new file.
4. Plot a graph of the analysis results.
5. Save the graph as an image, so we can put it in a paper.
6. Make a summary table of the analyses

Running `wordcount` and `plotcount.py` at the shell prompt, as we
have been doing, is fine for one or two files. If, however, we had 5
or 10 or 20 text files,
or if the number of steps in the pipeline were to expand, this could turn into
a lot of work.
Plus, no one wants to sit and wait for a command to finish, even just for 30
seconds. Also if we need to make a change to `wordcount`, we have to remember
the command we used to compile it.

One common solution to the tedium of data processing is to write
a shell script that runs the whole pipeline from start to finish.

The following script is an example of what you might do to automate some of 
this workflow:

~~~
# USAGE: bash run_pipeline.sh [need_to_compile]
# to produce plots for isles and abyss
# and the summary table for the Zipf's law tests

if [ $# -gt 0 ]; then
  if [ $1 = "need-to-compile" ]; then
    c++ --std=c++11 -o wordcount wordcount.cpp main.cpp
  fi
fi

./wordcount books/isles.txt > isles.dat
./wordcount books/abyss.txt > abyss.dat

python plotcount.py isles.dat isles.png
python plotcount.py abyss.dat abyss.png

# Generate summary table
python zipf_test.py abyss.dat isles.dat > results.txt
~~~
{: .bash}

This shell script solves several problems in computational reproducibility:

1.  It explicitly documents our pipeline,
    making communication with colleagues (and our future selves) more efficient.
2.  It allows us to type a single command, `bash run_pipeline.sh`, to
    reproduce the full analysis.
3.  It prevents us from _repeating_ typos or mistakes.
    You might not get it right the first time, but once you fix something
    it'll stay fixed.

Despite these benefits it has a few shortcomings.

Suppose we want to adjust the way the `plotcount.py` program works, such as
reducing the width of the bars in the plots.

Now we want to recreate our figures.

We _could_ just `bash run_pipeline.sh` again.
That would work, but it could also be a big pain if counting words takes
more than a few seconds.
The word counting routine hasn't changed; we shouldn't need to recreate
those files.

We have a few options in this case:

* We could manually rerun the plotting for each word-count file. (Experienced shell scripters can make this easier on themselves using a
for-loop.) With this approach, however, we don't get many of the benefits of having a shell script in the first place.
* We could comment out a subset of the lines in the script to not run the wordcount progams, then run the script again. But commenting out 
these lines, and subsequently uncommenting them, can be a hassle and source of errors in complicated pipelines.

Another problem is knowing if we need to recompile `wordcount`. How do we know
if `wordcount.cpp` or `main.cpp` have changed so that we need to add the `need-to-compile`
option to the script?

What we really want is an executable _description_ of our pipeline that
allows software to do the tricky part for us:
figuring out what steps need to be rerun.

Make was developed by
Stuart Feldman in 1977 as a Bell Labs summer intern, and remains in
widespread use today. Make can execute the commands needed to run our
analysis and plot our results. Like shell scripts it allows us to
execute complex sequences of commands via a single shell
command. Unlike shell scripts it explicitly records the dependencies
between files - what files are needed to create what other files -
and so can determine when to recreate our data files or
image files, if our text files change. Make can be used for any
commands that follow the general pattern of processing files to create
new files, for example:

* Run analysis scripts on raw data files to get data files that
  summarize the raw data (e.g. creating files with word counts from book text).
* Run visualization scripts on data files to produce plots
  (e.g. creating images of word counts).
* Parse and combine text files and plots to create papers.
* Compile source code into executable programs or libraries.

There are now many build tools available, for example [Apache
ANT][apache-ant], [doit][doit], and [nmake][nmake] for Windows. There
are also build tools that build scripts for use with these build tools
and others e.g. [GNU Autoconf][autoconf] and [CMake][cmake]. Which is
best for you depends on your requirements, intended usage, and
operating system. However, they all share the same fundamental
concepts as Make.

> ## Why Use Make if it is Almost 40 Years Old?
>
> Today, researchers working with legacy codes in C or FORTRAN, which
> are very common in high-performance computing, will, very likely
> encounter Make.
>
> Researchers are also finding Make of use in implementing
> reproducible research workflows, automating data analysis and
> visualisation (using Python or R) and combining tables and plots
> with text to produce reports and papers for publication.
>
> Make's fundamental concepts are common across build tools.
{: .callout}

[GNU Make][gnu-make] is a free, fast, well-documented, and very popular
Make implementation. From now on, we will focus on it, and when we say
Make, we mean GNU Make.

[autoconf]: http://www.gnu.org/software/autoconf/autoconf.html
[apache-ant]: http://ant.apache.org/
[cmake]: http://www.cmake.org/
[doit]: http://pydoit.org/
[gnu-make]: http://www.gnu.org/software/make/
[nmake]: https://msdn.microsoft.com/en-us/library/dd9y37ha.aspx
[zipfs-law]: http://en.wikipedia.org/wiki/Zipf%27s_law
