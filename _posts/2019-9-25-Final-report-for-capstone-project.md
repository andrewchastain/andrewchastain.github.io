---
layout: post
title: JHU-Coursera Capstone Project Final Report
author: Andrew Chastain
date: 8/28/2019
---
# Overview

The final project for the Data Science Specialization with Johns Hopkins University and Coursera was to develop a natural language processing (NLP) application that would determine the next word given an input string. This project was a collaboration with SwiftKey, the maker of a similar text prediction algorithm.  

## Natural Language Processing

Natural Language Processing (NLP) is the term used to describe the areas of study surrounding interpretting and using natural language data, especially as it relates to computerized analyses. One type of NLP model is an n-gram language model. N-gram language models are probabilistic models that use the frequencies of word combinations in a training set to calculate conditional probabilities for a word, given the words that precede it.  

### Maximum Likelihood Estimate

The simplest n-gram language model looks at bigram (word pair) frequencies to calculate a "maximum likelihood (ML) estimate." Given a bigram $\text{word}_1\;\text{word}_2$, the maximum likelihood estimate of $\text{word}_2$ is simply the frequency count of the bigram divided by the frequency of $\text{word}_1$. In mathematical notation this would be,  
$$
p_{ML}(\text{word}_2|\text{word}_1)=\frac{count(\text{word}_1\;\text{word}_2)}{count(\text{word}_1)}
$$  
This same model can be used with higher order n-grams by breaking the n-gram into a "root" (n-1)-gram (also known as a "history" by some authors) and a terminal word and calculating an ML estimate:  
$$
p_{ML}(w_i|w_{i-n+1}w_{i-n+2} \ldots w_{i-2}w_{i-1}) = p_{ML}(w_i|w_{i-n+1}^{i-1}) =   \frac{count(w_{i-n+1} \ldots w_{i-1}w_i)}{count(w_{i-n+1} \ldots w_{i-1})}
$$  
There are two main issues with this model. First, it assumes a zero probability for n-grams that are unseen in the training data. Second, it doesn't incorporate information from lower order n-grams, which is a way of incorporating additional meaning and context. The first issue can be solved using a "smoothing" method, where probability density from seen n-grams is reduced and redistributed. The second issue is able to be solved using methods like interpolation or back-off.

### Kneser-Ney Probability

One method that has been shown to more accurately calculate the terminal word probability is the Kneser-Ney algorithm. This algorithm corrects the above issues by employing a "discounting" of the probability of infrequent words, and adjusting the final probability by calculating a "continuation probability" for the final word that takes into consideration the number of different possible terminals for the root, and roots for the terminal word. The algorithm will be discussed more, further on.

## Cleaning the training data

The training data that was provided by SwiftKey consisted of 899,288 fragments of blog posts, 1,010,242 fragments of news articles, and 2,360,148 tweets. While these sources were primarily in English, the use of different characters (such as emoticons in Twitter), smart punctuation (typically from Microsoft Word) and occasional foreign words or characters complicated the analysis and processing. Furthermore, encoding issues between UTF-8 and Windows-1252 would cause many of those character to display in unicode or break a character into a set of incorrect unicode characters. Stripping out just those characters (or words containing those characters) had the consequence of breaking context.  

For example, if a string read "I saw it in the Encyclopædia Britannica" and the æ character was encoding incorrectly then removing the character would create the word "Encyclopdia" (which increases number of unique n-grams, but for a word that wouldn't be seen except in error). Removing the word "Encyclopædia" would break the context as well, as it would generate n-grams that weren't in the original text ("in the Britannica"). Both situations would generate spurious results. Thus, modifying the text during cleaning had a significant risk of generating bad n-grams and impacting the probability calculations. While there are other techniques, such as stemming or crafting elaborate replacement strategem, those would have taken too much time to implement in this case.

### Unicode Scripts and Categories

The workaround that I employed for these problematic characters was to filter out sentences that contained problematic characters. I did this primarily using the unicode scripts and categories. Unicode scripts refers to the character sets that are assigned to specific languages. An example of a unicode script is `\\p{Thai}`, which is the collection of all unicode characters for rendering Thai. Categories are groups of characters that are related in function, such as `\\p{N}`, which represents all numbers or `\\p{Cf}` which are invisible formatting indicators. From the list at [regular-expressions.info](http://www.regular-expressions.info/unicode.html) I made a scripts list:  


```r
scripts <- list("\\p{L}", "\\p{Ll}", "\\p{Lu}", "\\p{Lt}", "\\p{L&}",
                "\\p{Lm}", "\\p{Lo}", "\\p{M}", "\\p{Mn}", "\\p{Mc}",
                "\\p{Me}", "\\p{Z}", "\\p{Zs}", "\\p{Zl}", "\\p{Zp}",
                "\\p{S}", "\\p{Sm}", "\\p{Sc}", "\\p{Sk}", "\\p{So}",
                "\\p{N}", "\\p{Nd}", "\\p{Nl}", "\\p{No}", "\\p{P}",
                "\\p{Pd}", "\\p{Ps}", "\\p{Pe}", "\\p{Pi}", "\\p{Pf}",
                "\\p{Pc}", "\\p{Po}", "\\p{C}", "\\p{Cc}", "\\p{Cf}",
                "\\p{Co}", "\\p{Cs}", "\\p{Cn}",
                "\\p{Common}", "\\p{Arabic}", "\\p{Armenian}",
                "\\p{Bengali}", "\\p{Bopomofo}", "\\p{Braille}",
                "\\p{Buhid}", "\\p{Canadian_Aboriginal}", "\\p{Cherokee}",
                "\\p{Cyrillic}", "\\p{Devanagari}", "\\p{Ethiopic}", 
                "\\p{Georgian}", "\\p{Greek}", "\\p{Gujarati}", 
                "\\p{Gurmukhi}", "\\p{Han}", "\\p{Hangul}", "\\p{Hanunoo}", 
                "\\p{Hebrew}", "\\p{Hiragana}", "\\p{Inherited}", 
                "\\p{Kannada}", "\\p{Katakana}", "\\p{Khmer}", "\\p{Lao}", 
                "\\p{Latin}", "\\p{Limbu}", "\\p{Malayalam}", 
                "\\p{Mongolian}", "\\p{Myanmar}", "\\p{Ogham}", 
                "\\p{Oriya}", "\\p{Runic}", "\\p{Sinhala}", "\\p{Syriac}", 
                "\\p{Tagalog}", "\\p{Tagbanwa}", "\\p{Tamil}", 
                "\\p{Telugu}", "\\p{Thaana}", "\\p{Thai}", "\\p{Tibetan}", 
                "\\p{Yi}")
```

I was then able to count how many lines had example of each class (in the blogs, in the code example below):  


```r
dt <- data.table("script" = scripts, "count" = NA)
i = 1
while(i < 83) {
    dt[[i,2]] <- sum(grepl(dt[[i,1]], blogs, perl = T))
    i <- i + 1
}
```

Since numbers, non-Latin/Common scripts, and weird punctuation/emoticons/control characters were uncommon and there were a large number of documents to use, it was fastest to filter out anything that would otherwise need to be cleaned. That way only "clean" sentences would be used and context for those sentences would be maintained.

### Additional Cleaning

After removing the majority of the bad characters there were still some punctuation and common characters that needed to be removed.  

Finally, the corpus comprised of the three sources was run through the function `clean2()` via the `clean_wrapper()` function. This function removed text based emoticons (for example, :-) or :D), coverted smart and alternative apostrophes to ', and handled more complex conversions like removing periods used in abbreviations and removing non-internal apostrophes (remove apostrophes from 'this' but not from isn't). 

### Testing and Validation Sets

After the data was cleaned it was split into testing and validation sets. This allowed for checking the accuracy of the model.

## N-gram generation

While libraries like `quanteda` and `tm` were tested in early development, it appeared to be faster and cleaner to convert the "documents" to sentences using strsplit with the regular expression `"(?<=\\.)\\s(?=[a-z<])"`.  It was important to split into sentences to prevent the n-gram generating function from spanning periods and generating spurious n-grams.  

To get around limitation in R with regards to memory size, the set of all sentences was split into ten fragments which could then individually be "tokenized" (split into individual words) then recombined into n-grams. The n-grams were made by taking a window of length n, pasting together the words in the window and sliding it along each sentence. Thus each sentence was converted to a list of n-grams. The raw lists of n-grams from each fragment were then grouped and counted, utilizing the speed of `data.tables`. These fragments were then recombined to produce frequency lists for n-grams of length 2 to 6.  

The methodology of splitting with strsplit and using a sliding window for n-gram generation was taken from the `cmscu` package tutorial from [Dave Vinson](https://github.com/DaveVinson/cmscu-tutorial/blob/master/cmscu-tutorial.Rmd). This method avoided the process of creating a document-term matrix and subsequent frequency table that is utilized in `quanteda`.

## The Modified Kneser-Ney Algorithm

Once the corpus was converted to a set of n-grams and the counts of their occurances in the corpus the modified Kneser-Ney smoothing algorithm could be used to calculate the probabilities of each terminal word.  

The modified Kneser-Ney smoothing algorithm hinges on calculating a reduction (a discount) for each measured frequency (similar to Good-Turing discounting), and then moving the lost probability around to account for the uniqueness of a given n-gram. This uniqueness is measured as a "continuation probability," which counts the number of different "histories" a word could have, divided by the total number of unique combinations. Given an n-gram $w_{i-n+1}^{i}=w_{i-n+1}w_{i-n+2} \ldots w_{i-1}w_i$ the probability of $w_i$, given $w_{i-n+1} \ldots w_{i-1}$ (or $w_{i-n+1}^{i-1}$) can be calculated as:  
$$
p_{KN}(w_i|w_{i-n+1}^{i-1}) = \frac{c(w_{i-n+1}^i) - D(c(w_{i-n+1}^i))}{\sum_{w_i} c(w_{i-n+1}^i)} + \gamma(w_{i-n+1}^{i-1}) p_{KN}(w_i|w_{i-n+2}^{i-1})
$$  
In the above equation $c(w_{i-n+1}^i)$ refers to the count of the n-gram, $D(c(w_{i-n+1}^i))$ refers to the discount function, which depends on the count, $\gamma(w_{i-n+1}^{i-1})$ is the interpolation function that calculates how much of the probability distribution was removed by the discounting, and $p_{KN}(w_i|w_{i-n+2}^{i-1})$ is the "continuation probability". Each will be discussed further below.

### The discounting modification  

Kneser-Ney smoothing was originally formulated to have a single discounting that could be determined from a held-out set of data. As indicated by Chen and Goodman (1999), the performance of the algorithm was improved by creating three different $D(c)$ values. These values are:  
$$
D(c)= \begin{cases}
0 & \text{if }\; c = 0 \\
D_1 & \text{if }\; c = 1 \\
D_2 & \text{if }\; c = 2 \\
D_{3+} & \text{if }\; c \geq 3
\end{cases}
$$  
where  
$$
\begin{align}
Y &= \frac{n_1}{n_1 + 2n_2} \\
D_1 &= 1 + 2Y\frac{n_2}{n_1} \\
D_2 &= 2 + 3Y\frac{n_3}{n_2} \\
D_{3+} &= 3 + 4Y\frac{n_4}{n_3}
\end{align}
$$  

The values of $n_1$ to $n_4$ come from the frequency distribution of counts, such that $n_1$ is the number of n-grams that occur only once in the corpus, $n_2$ occur twice, etc. The derivation for the D values is indicated in the Chen and Goodman paper as coming from Klaus Ries in 1997 in personal communications.  

### Gamma, the interpolation function  

The function $\gamma(w_{i-n+1}^{i-1})$ determines the amount of the probability distribution that has been removed by the discounting $D(c)$. In the Chen and Goodman paper this is written as:  
$$
\gamma(w_{i-n+1}^{i-1}) = \frac{D_1 N_1(w_{i-n+1}^{i-1}\:\bullet) + D_2 N_2(w_{i-n+1}^{i-1}\:\bullet) + D_{3+} N_{3+}(w_{i-n+1}^{i-1}\:\bullet)} {\sum_{w_i} c(w_{i-n+1}^i)}
$$  
where  
$$
N_r(w_{i-n+1}^{i-1}\:\bullet) = |\{w_i:c(w_{i-n+1}^{i-1}w_i)=r \}|
$$  

In other words, $N_1$ is the count of different words $w_i$ that terminate the root $w_{i-n+1}^{i-1}$ only once. $N_2$ is the count of terminal words that occur twice in training.  

### The continuation probability  

The continuation probability is a lower-order probability distribution that seeks to calculate how "predictive" the start of the n-gram is. The continuation probability is written as such:  

$$
p_{KN}(w_i|w_{i-n+2}^{i-1}) = \frac{N_{1+}(\bullet\: w_{i-n+2}^i)}{N_{1+}(\bullet\: w_{i-n+2}^{i-1} \:\bullet)}
$$  
where
$$
\begin{align}
N_{1+}(\bullet\: w_{i-n+2}^i) &= |\{w_{i-n+1}:c(w_{i-n+1}^i)>0\}| \\
N_{1+}(\bullet\: w_{i-n+2}^{i-1} \:\bullet) &= |\{(w_{i-n+1},w_{i}):c(w_{i-n+1}^i)>0\}| = \sum_{w_i} N_{1+}(\bullet\: w_{i-n+2}^i) \\
\end{align}
$$

Following the earlier syntax, the numerator is the count of unique words $w_{i-n+1}$ that can precede $w_{i-n+2} \ldots w_i$ while the denominator is the sum of those counts for all words $w_i$.  

## Backoff Strategy

One difference between this implementation of the modified Kneser-Ney Smoothing and the one presented in the Chen and Goodman paper is that this implementation treats all n-grams as the highest order n-gram. In the 1999 paper implementations, in sections 4.1.6-7, n-gram levels below the highest are smoothed by using the unique combination counts instead of corpus frequency:  
$$
p_{KN}(w_i|w_{i-n+1}^{i-1}) = \frac{N_{1+}(w_{i-n+1}^i) - D(N_{1+}(w_{i-n+1}^i))}{\sum_{w_i} N_{1+}(w_{i-n+1}^i)} + \gamma(w_{i-n+1}^{i-1}) p_{KN}(w_i|w_{i-n+2}^{i-1})
$$  
It was not feasible to develop the lower-order model originally, but it would be interesting to compare the models in this task.  

The brief for this project asked for an application to produce a single word prediction based on an input string. Once $p_{KN}$ values were calculated for each root-term pair it was possible to discard all but the highest probability terminal word. This greatly reduced the size of the database needed to encode a reasonable number of known inputs for each n-gram level from a 5-gram (a 6-gram broken into a root and prediction) down to a unigram (split bigrams) model.  

By calculating the $p_{KN}$ and creating a master look-up table ahead of time, the Shiny.io application was able to be very simple - after cleaning the input string it would look up the last five words. If it found a prediction, it would return it; otherwise, it would chop off the first word and run it again. If it recursed down to the unigram model and didn't find a match, it would return "the," the most common word in the corpus. 
