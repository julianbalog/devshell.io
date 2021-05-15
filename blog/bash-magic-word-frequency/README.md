Bash Magic - Word Frequency
===========================

_Calculating the frequency of each word in a regular text file using Bash_

https://devshell.io/bash-magic-word-frequency/

In this post we'll look at Bash and its somewhat unique ability to provide easy solutions to relatively complex problems. Let's consider the classic problem of calculating the frequency of each word in a regular text file, `input.txt`. Words can appear in uppercase or lowercase letters. We assume that words are separated by any number of whitespaces.

Given the following input text, for example:

```
docker Kubernetes  
  AWS  kubernetes aws
    Docker aws  
EKS   eks Azure
```

We expect the following output:

```
aws 3
kubernetes 2
eks 2
docker 2
azure 1
```

With Bash, we can write a simple one-liner producing the aforementioned result. Here's the related command:

```
tr '[:upper:]' '[:lower:]' < input.txt | tr -s ' ' '\n' | sort | uniq -c | sort -r | awk '{print $2, $1}'
```

We'll look at the anatomy of the command next.

The command explained
---------------------

Let's lay out our command into several lines for easier reference:

```
tr '[:upper:]' '[:lower:]' < input.txt | \
tr -s ' ' '\n' | sort | \
uniq -c | sort -r | \
awk '{print $2, $1}'
```

Line 1 transforms the input text to contain lowercase letters only:

```
tr '[:upper:]' '[:lower:]' < input.txt
```

Line 2 extracts the words based on the whitespace separator `' '`, prints each word on a new line and sorts the output:

```
tr -s ' ' '\n' | sort 
```

The resulting output is:

```
aws
aws
aws
azure
docker
docker
eks
eks
kubernetes
kubernetes
```

Line 3  consolidates the adjacent identical lines into unique entries and prefixes the result with the number of occurences for each. The output is then sorted in reverse order:

```
uniq -c | sort -r
```

The output is:

```
3 aws
2 kubernetes
2 eks
2 docker
1 azure
```

Finally, line 4 swaps the two parts of each line, with the word first and the number of occurrences next.

```
awk '{print $2, $1}'
```

We get our final output as:

```
aws 3
kubernetes 2
eks 2
docker 2
azure 1
```

This solution may not be the most efficient one, but it's relatively simple and gets the job done without extensive coding.
