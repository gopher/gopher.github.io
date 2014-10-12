---
layout: post
title: Basic IO in Go
categories:
- blog
---

One task I do a lot in my day job - when not answering emails that
is - is analyzing all sorts of scientific data. Often, this data consists of
small to medium sized plain text files that need to be processed to count certain
elements and do basic statistics on them. Go turns out to be a fantastic
language to do all this and much more in an efficient manner.

To warm up, we'll write a small program which will count the number of
characters and words contained in an arbitrary text file. This is similar to
what the popular wc program available on must Linux and other Unixy
operating systems does. Sounds easy enough.

Before we get started we need to clarify what we mean by characters
and words. Since I happen to mostly deal with ASCII data I frequently
tend to neglect the fact that when processing textual information we really
ought to use unicode. If you don't know what unicode is, stop right here and
head over to [this post](http://www.joelonsoftware.com/articles/Unicode.html).
Go has excellent unicode support in the form of UTF-8 and Go source code itself
is UTF-8 encoded. Since ASCII is a proper subset of UTF-8, our unicode aware
wordcount program will thus get us ASCII parsing for free so I'm happy.
That out of the way, let's go back to characters and words. In the unicode
spirit, we define a character as a unicode code point represented in Go as
type *rune*, which is an alias for type int32. Much more information regarding
strings and unicode is available in [this](http://blog.golang.org/strings) great
blog post. A word, then, is a sequence of runes and individual words in a
piece of text are sequences of runes separated by whitespace.

Now to some actual code: We start with an initial Go command line program that
will accept the name of text file, open the file, and then
read and print its content to standard out. Once this is in place we can
then add additional processing code for counting characters and words.

{% highlight go linenos %}
package main

import (
  "bufio"
  "fmt"
  "log"
  "os"
)

func main() {
  if len(os.Args) != 2 {
    log.Fatal("usage: wc <filename>")
  }

  fileName := os.Args[1]
  file, err := os.Open(fileName)
  if err != nil {
    log.Fatal(err)
  }
  defer file.Close()

  scanner := bufio.NewScanner(file)
  for scanner.Scan() {
    fmt.Println(scanner.Text())
  }
  if scanner.Err() != nil {
    log.Fatal(err)
  }
}
{% endhighlight %}

I hope this piece of code is straighforward even to those of you who haven't
been exposed to Go very much - or at all.

 * __[11-13]__ Within *main*, our code first checks if a filename was provided.
 Keep in mind that the name of the executable itself is the first entry in the
 list of command line arguments.

 * __[15-20]__ Next, we open the file. If we encounter an error we terminate and
 log the error to the console. Using Go's defer mechanism we ensure that the
 file is closed once the function exits.

* __[22-28]__ Here's where the real action happens: We use a
[bufio.Scanner](http://golang.org/pkg/bufio) provided by Go's
standard library to walk through the file. Whenever you need
to process textual content a *bufio.Scanner* should be on the top
of your list of things to try. The call to *scanner.Scan()* in the for loop
advances to the next available token and returns true if one is available and
false otherwise. The latter occurs on end-of-file (EOF) or earlier
if an error is encountered. Inside the for loop we then simply print a string
representation of the scanned token via a call to *scanner.Text()*. Once we're
done scanning we check if the scanner encountered any errors, and if yes, we
terminate and log them.

The concept of a token requires further clarification. In fact, what
makes a *bufio.Scanner* so useful is that we can provide our own prescription
of what a token is and *bufio.Scanner* will happily use it to scan textual
information for us. For those who don't want to bother with defining their own
token recipe, *bufio* provides several prescribed ones: *ScanBytes*, *ScanRunes*,
*ScanLines*, and *ScanWords* for scanning byte, rune, line or word tokens. To
instantiate a new token scanner on an existing *bufio.Scanner* simply call the
Split() function with the new recipe as argument.
The default token recipe is *ScanLines* so we would expect that our
code above will print the file content line by line. Let's try
it out.

Create a directory **wordcount/** somewhere and save the code above into a file
named **wordcount.go**. Then compile via

{% highlight bash %}
# go build
{% endhighlight %}

Now, let's grab a piece of text from [Project Gutenberg](http://www.gutenberg.org/).
I've chosen Charles Dicken's Bleak House, but you are welcome to use any other
piece of text of your choice.

{% highlight bash %}
# wget http://www.gutenberg.org/ebooks/1023.txt.utf-8
{% endhighlight %}

Great. Finally we are ready to try out the first version of our *wordcount*
program

{% highlight bash %}
# ./wordcount 1023.txt.utf-8
{% endhighlight %}

If everything worked out as expected you should see the content of Bleak House
zoom past your screen. Congratulations! If not, welcome to another important
aspect of software development, debugging.

With this file parsing framework in place, we are in pretty good shape to
add our character and word counting code to complete the wordcount program.
In fact, if we wanted to count either characters or words we would be basically
done since we could just use *bufio's* *ScanRune* or *ScanWord* token recipe,
respectively. But we said that we wanted to count both characters and words
simultaneously so we'll have to do a bit more work. A simple solution would be
to read through the file twice using first *ScanRune* and then *ScanWord*
the second time around. For small files this might in fact be a satisfactory
solution and since it only utilizes standard library functions would be easy to
implement correctly. However, to make our wordcount program as efficient as
possible we will insist on parsing the underlying text file only once.

Here's a possible implementation.

{% highlight go linenos %}
package main

import (
  "bufio"
  "fmt"
  "log"
  "os"
  "unicode"
  "unicode/utf8"
)

func main() {
  if len(os.Args) != 2 {
    log.Fatal("usage: wc <filename>")
  }

  fileName := os.Args[1]
  file, err := os.Open(fileName)
  if err != nil {
    log.Fatal(err)
  }
  defer os.Close(file)

  scanner := bufio.NewScanner(file)
  scanner.Split(bufio.ScanRunes)

  var runeCount int
  var wordCount int
  inWord := false
  for scanner.Scan() {
    b := scanner.Bytes()
    r, _ := utf8.DecodeRune(b)

    runeCount++
    if unicode.IsSpace(r) && inWord {
      inWord = false
      wordCount++
    } else if !unicode.IsSpace(r) && !inWord {
      inWord = true
    }
  }
  if scanner.Err() != nil {
    log.Fatal(err)
  }

  fmt.Printf("%s:  #chars %d    #words %d\n", fileName, runeCount, wordCount)
}
{% endhighlight %}

As you can see, with only about 10 lines of additional code we have a fully
functional wordcount program.

* __[25]__ Since we need to count individual unicode code points, runes in Go
parlance, we use *bufio.ScanRunes* as token scanner.

* __[27-29]__ We initialize two *int* counter variables which track the number
of runes and words in the file. *inWord* is a boolean variable indicating
if the current scan position within the file is within a word token or not.

* __[30-44]__ Inside the for loop we extract the current scanned token as a byte
slice and decode it as a rune using *utf8.DecodeRune*. If b would be any byte
slice we would also need to check if the call to *utf8.DecodeRune* has been
successful. However, *bufio.ScanRunes* will already ensure that b is a valid
rune (otherwise the for loop will terminate and we'll catch the error in line
42-44) and we can skip this check. At this point in the for loop we successfully
read the next rune and we can increment our runeCount variable. The code block
in lines 35-40 tracks if we are currently inside a word or not. Each time we
leave the current word we increment wordCount by one.

* __[46]__ We end by printing the character and word count to standard out.

Voila, this concludes our shiny new wordcount program. Or so we hope to
think. No software development project is complete without proper testing,
testing, and some more testing. As a simple test, let's compare our wordcount
against the Unix wc program.

{% highlight bash %}
# wc 1023.txt.utf-8
   40234  356465 2001189 1023.txt.utf-8
{% endhighlight %}

So, wc tells us that Charles Dicken's Bleak House contains 356465 words and
2001189 characters (the first number, 40234, is the number of lines the file
has). Now our wordcount

{% highlight bash %}
# ./wordcount 1023.txt.utf-8
1023.txt.utf-8:  #chars 2001187    #words 356465
{% endhighlight %}

Looking great, or wait. Not quite. Our wordcount returns the same number of words,
356465, as wc, but the number of characters is off by two. Who's correct?

Interestingly, the answer in this case is that both wc and our wordcount are correct.
The reason for the discrepancy is that our definition of a character (a unicode
code point or rune) is different from wc's which simply counts the number of
bytes (ASCII characters) in the file. If we look carefully at *1023.txt.utf-8*
we find that the first code point in the file is the
[unicode byte order mark](https://en.wikipedia.org/wiki/Byte_order_mark) (BOM)
*U+FEFF* which is three bytes wide. Our wordcount program counts the BOM as
a single rune, while wc will clock it as 3 separate characters or bytes. This is
exactly where the difference in 2 in the character count comes from. As a
corollary, for counting unicode code points in utf8 encoded text files,
the standard wc utility probably is not the best tool for the job.

This concludes our initial tour of Go's powerful IO facilities. Congratulations
on your new wordcount utility. In the next post we will take a more in depth
look at all things IO.












