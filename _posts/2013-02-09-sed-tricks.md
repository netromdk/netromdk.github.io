---
layout: post
title:  "sed tricks"
date:   2013-02-09 16:05:04 +0200
---

The <strong>s</strong>tream <strong>ed</strong>itor, most commonly know as <b>sed</b>, is a wonderful tool for modifying data from files and stdin. <em>In this article I will be using the BSD variant of sed which is a little bit different from the GNU variant of sed but I will try to point out where the differences are in my examples.</em> 

<h4>Usage</h4>
One of the most common ways of using sed is:

```shell
cat file.txt | sed COMMAND
```

And the other

```shell
sed COMMAND FILE
```

It is possible to have multiple commands using the `-e` argument:

```shell
sed -e COMMAND -e COMMAND ..
```

Or even like this:

```shell
sed 'COMMAND;COMMAND;..'
```

The <em>COMMAND</em> can be a lot of different things but usually it will be the substitution pattern: `s/regular expression/replacement/flags`

Here is an example:

```shell
echo "aabb" | sed -e 's/a/A/' -e 's/b/B/g'
AaBB
```

Notice that the first command is only run on the first occurrence and the second is run on all using the `g` flag. Keep in mind that it works on a <em>line-by-line</em> basis. 

Another important aspect is <em>in-place</em> alteration of files:

```shell
sed -i (EXT) COMMAND FILE
```

This command will edit FILE using COMMAND and if the extension EXT is given then a backup is saved to the FILE with EXT appended to the filename. It is generally recommended to produce backup files so nothing is lost unintentionally.

<h4>Group matching</h4>
Often it is necessary to match some block of data and substitute some portions without removing other parts, or to alter the order of blocks. A group is matched using parentheses in the command and referenced with `\\1`,`\\2`.. etc., for instance:

```shell
echo "foobar" | sed 's/\\(foo\\)\\(bar\\)/\\2\\1/'
barfoo
```
Two groups are matched ("foo" and "bar") and their order is reversed. Notice that the parentheses have to be escaped in order to prevent matching the actual characters "(" and ")".

<h4>Extended regular expressions</h4>
The expressions I have used up until now were basic regular expressions but a more powerful variant exists, namely the extended regular expressions. In this mode, along with a lot of stuff, it's not necessary to escape parentheses and the POSIX character sets are available. Example:

```shell
echo "foo \\t bar" | sed -E 's/(foo)[[:space:]]*(bar)/\\2 \\1/'
bar foo
```
The character set `[:space:]` matches any whitespace character. <b><em>`-E` is replaced with `-r` in GNU sed.</em></b>

<a href="http://www.grymoire.com/Unix/Regular.html" target="_blank">Here</a> is a great reference of the different regular expressions, both basic and extended ones.

<h4>Case sensitivity</h4>
In GNU sed there is a flag to turn off case sensitivity, namely the "i" flag. Sadly this flag is not available in BSD sed so one has to turn to other possibilities. One is to decapitalize all letters before piping to sed, but that doesn't work for files and in-place modifications. However, case can be ignored when using character sets and similar constructs so keep that in mind, i.e. `[:alpha:]` will match "A" as well as "a". Selective parts of a regular expression can sometimes require specific characters so if one wants to match both cases it can be done using ranges, like `[a-cA-C]` for instance.

Different approaches exist for decapitalizing using pipes. Here is one using `tr`:

```shell
echo "Hello, World" | tr '[:upper:]' '[:lower:]'
hello, world
```

And here is one using `awk`:

```shell
echo "Hello, World" | awk '{print tolower($0)}'
hello, world
```

If all else fails Perl has a sed-like substitution syntax  that accepts the "i" flag:

```shell
echo "HeLlo" | perl -pe 's/l/./ig'
He..o
```

<h4>Greedy vs non-greedy matching</h4>
sed is greedy by default, meaning it will try to match as much as possible when using `+` or `*`. Here's a greedy example:

```shell
echo "aaa(bbb)aaa(bbb)aaa" | sed -E 's/\\(.*\\)/./'
aaa.aaa
```
The above matches a "(" then anything greedily until next ")". Suppose we wanted to only match the first "(bbb)" and not "(bbb)aaa(bbb)", then we could do the following:

```shell
echo "aaa(bbb)aaa(bbb)aaa" | sed -E 's/\\([^\\(]*\\)/./'
aaa.aaa(bbb)aaa
```
As before it matches a "(" then matching non-greedily for "(" (meaning <em>not</em> matching a "(") until we match the closing ")".

<h4>Examples</h4>
A friend of mine recently asked me how to replace entries of the form "&lt;email address&gt;" with "&lt;--removed--&gt;" using sed. The following was the solution:

```shell
cat file.txt | sed -E 's/(<).*@.*(>)/\\1--removed--\\2/g'
```
It's not a correct regexp for matching email addresses but given the knowledge of the presence of &lt;, &gt; and @ it was fitting in his scenario.

The proper way of doing it would be the following:

```shell
cat file.txt | sed -E 's/(<)[[:alpha:][:digit:]\\._%\\+-]+@[[:alpha:][:digit:]\\.-]+\\.[[:alpha:]]{2,4}(>)/\\1--removed--\\2/g'
```

Another useful thing is to escape spaces in filenames and use them with commands:

```shell
#!/bin/sh
DIR=`echo $1 | sed 's/ /\\\\\\\\ /g'`
eval echo "Listing ${DIR}:"
eval ls -l ${DIR}
```

I used <em>double-escaped backslashes</em> because the final output should be, for instance, "/test\\ one\\ seven/foobar.txt" and not "/test one seven/foobar.txt" so that the commands can interpret the paths correctly (using the `eval` command). The effect is that the script can be called with both an escaped or non-escaped path, i.e. `"/test one seven"` or `/test\\ one\\ seven`.

When creating a Linux distribution of a program or similar that is required to run off-the-bat, and where the source code might not be available, it is often needed to get all the shared library dependencies of certain binary files. The following script will copy these to the destination directory: <b>(Linux only!)</b>

```shell
#!/bin/sh
# Usage: getshared.sh <binary> <dest directory>
BIN=`echo $1 | sed 's/ /\\\\\\\\ /g'`
DST=`echo $2 | sed 's/ /\\\\\\\\ /g'`
eval mkdir -p ${DST}
eval ldd ${BIN} | grep "=>" | sed 's/.*\\s*=>\\s*\\(.*\\)(.*)/\\1/' | awk '{print $1};' | eval xargs -I{} cp -vuL {} ${DST}
```

Here the sed command retrieves the dependency library from the output using the `\\1` group. A sample line from `ldd` could be

```shell
libc.so.6 => /usr/x86_64-linux-gnu/libc.so.6 (0x00007f6eb8c000)
```

Assume we have a file we want to modify in-place, like an INI configuration file like this:

```ini
[Section]
enableStuff=false  
someVar=1337
enableOther = false
dontTouchThis="wicked"
oldSchool: false
```

And let's say we wanted to replace all instances of "false" with "true":

```shell
sed -i .bck -E 's/([[:alnum:]]+)[^\\s]*([=:])[^\\s]*false/\\1\\2true/' conf.ini
```

You will get the following result in `conf.ini` (along with the backup in `conf.ini.bck`):

```ini
[Section]
enableStuff=true  
someVar=1337
enableOther=true
dontTouchThis="wicked"
oldSchool:true
```

Notice that I remove all whitespace and that I allow both "=" and ":" as the delimiter because some implementations use ":" however uncommon it may be.

That concludes my tips and tricks using sed.

