Using Lemmatization and POS tagging
===================================

_(C) 2014 Wouter van Atteveldt, license: [CC-BY-SA]_

For Dutch, rule-based stemming does not work very well. Consider:


```r
library(RTextTools)
```

```
## Loading required package: SparseM
## 
## Attaching package: 'SparseM'
## 
## The following object is masked from 'package:base':
## 
##     backsolve
```

```r
library(slam)
text = "De verzekeringen zijn betaald, ik betaal de verzekering"
col_sums(create_matrix(text, language="dutch", stemWords=T, removeStopwords=F))
```

```
##  betaald    betal verzeker     zijn 
##        1        1        2        1
```

As you can see, _verzekering_ and _verzekeringen_ are both reduced to _verzeker_, _betalen_ and _betaald_ are kept separate. 
By comparison, a proper lemmatizer will handle such cases properly.
For example, the Frog lemmatizer accessible at http://ilk.uvt.nl/cgntagger/ produces the following output for that sentence (use the text box on the right)

sent. # | word # |  word |  tag |  lemma
:-------------|:------------|:-------------|:------------|:-------------
0  | 0  |  De |  LID(bep,stan,rest) |  de
0  | 1  |  verzekeringen |  N(soort,mv,basis) |  verzekering
0  | 2  |  zijn |  WW(pv,tgw,mv) |  zijn
0  | 3  |  betaald |  WW(vd,vrij,zonder) |  betalen
0  | 4  |  , |  LET() |  ,
0  | 5  |  ik |  VNW(pers,pron,nomin,vol,1,ev) |  ik
0  | 6  |  betaal |  WW(pv,tgw,ev) |  betalen
0  | 7  |  de |  LID(bep,stan,rest) |  de
0  | 8  |  verzekering |  N(soort,ev,basis,zijd,stan) |  verzekering

The final column contains the _lemma_, and as you can see the different verb conjugations are now matched together. You can also see the 'tag' column, which lists the part of speech (POS), indicating what kind of word it is. It is often very useful to select e.g. only nouns, adjectives, and verbs. 

Lemmatizing text
----

Performing actual lemmatization is beyond the scope of this document. 
Frog can be installed easily using `apt-get install frog frogdata ucto` on a modern debian or ubuntu system (see also http://ilk.uvt.nl/frog/), and for most common languages free lemmatizers are available. 

For this workshop, I've lemmatized the 'achmea.csv' documents using the frog lemmatizer built into AmCAT, and exported the tokens as a data frame. This file can be downloaded from [github](https://github.com/vanatteveldt/learningr/blob/master/tokens.rdata?raw=true)


```r
load("tokens.rdata")
head(tokens)
```

```
##         aid achmea_id     word  lemma pos1 freq
## 1 112028081         1     Snel   snel    A    1
## 2 112028081         1 geholpen helpen    V    1
## 3 112028081         1       ..     ..    .    2
## 4 112028081         1   Gevoel gevoel    N    1
## 5 112028081         1      dat    dat    C    1
## 6 112028081         1       ik     ik    O    1
```

As you can see, this data frame contains the pos, word and lemma columns from the table above. 
The _pos1_ columns contains an abbreviated POS tag, which is easier to use. 

From tokens to document-term matrix
----

Although tokens are in principle the same 'triplet' representation as a sparse matrix, 
there are a number of steps to move from tokens to a document-term matrix.
We define a function `dtm.create` here, which is also available [corpustools](http:/github.com/kasperwelbers/corpustools) package.


```r
library(tm)
```

```
## Loading required package: NLP
```

```r
library(Matrix)
cast.sparse.matrix <- function(rows, columns, values) {
  d = data.frame(rows=rows, columns=columns, values=values)
  d = aggregate(values ~ rows + columns, d, FUN='sum')
  unit_index = unique(d$rows)
  char_index = unique(d$columns)
  sm = spMatrix(nrow=length(unit_index), ncol=length(char_index),
                match(d$rows, unit_index), match(d$columns, char_index), d$values)
  rownames(sm) = unit_index
  colnames(sm) = char_index
  sm
}

dtm.create <- function(ids, terms, freqs) {
  # remove NA terms
  d = data.frame(ids=ids, terms=terms, freqs=freqs)
  d = aggregate(freqs ~ ids + terms, d, FUN='sum')
  id_index = unique(d$ids)
  term_index = unique(d$terms)
  sm = spMatrix(nrow=length(id_index), ncol=length(term_index),
                match(d$ids, id_index), match(d$terms, term_index), d$freqs)
  rownames(sm) = id_index
  colnames(sm) = term_index
  as.DocumentTermMatrix(sm, weighting=weightTf)
}
```

The following code creates a document-term matrix containing only the nouns


```r
nouns = tokens[tokens$pos1 == 'N', ]
m = dtm.create(nouns$achmea_id, nouns$lemma, nouns$freq)
class(m)
```

```
## [1] "DocumentTermMatrix"    "simple_triplet_matrix"
```

```r
dim(m)
```

```
## [1] 20490 13050
```

Since this is a 'normal' DocumentTermMatrix, it can be used for corpus analysis, topic modeling or machine learning like the matrices created using the `create_matrix` function from `RTextTools`

