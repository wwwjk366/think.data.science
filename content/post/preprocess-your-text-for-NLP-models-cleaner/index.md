---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Preprocess Text in Python --- A Cleaner and Faster Approach"
subtitle: ""
summary: ""
authors: ['admin']
tags: ['Python', 'NLP', 'Parallel Computing']
categories: ['Blog', 'Tutorial']
date: 2019-07-04T20:14:43Z
lastmod: 2019-07-04T20:14:43Z
featured: true
draft: false

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: ""
  focal_point: ""
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---



## Motivation

Well, I think it all start with one of my favorite tweets from 2013:


<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">In Data Science, 80% of time spent prepare data, 20% of time spent complain about need for prepare data.</p>&mdash; Big Data Borat (@BigDataBorat) <a href="https://twitter.com/BigDataBorat/status/306596352991830016?ref_src=twsrc%5Etfw">February 27, 2013</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>



When building NLP models, pre-processing your data is extremely important. For example, different stopwords removal, stemming and lemmization might have huge impact on the accuracy of your models. Often times, the order of how you do the cleaning is also critical. Do you want to remove certain words first then tokenize the text? Or tokenize then remove the tokens? What we need is a **clear to understand** and yet **flexiable** code to do the pre-processing job. When using R, the pipe operator `%>%` kind of taken care of the most part. However, there is no really good equivlent in Python because the natural different of Python and R: [(Long but very good read)](https://medium.com/@jondot/functional-programming-with-python-for-people-without-time-1eebdbd9526c)

> In other words, it’s like saying that when OOP was born, it was also born with the Gang-of-Four design patterns baked into it’s core as its backing theory of thought (outside of types and inheritance and methods etc.), and every implemented OOP language included these patterns and abstractions by default for you to take advantage of, and that these patterns were bullet-proofed by centuries of research. But that can already never be correct — the Singleton pattern is by now widely recognized as an anti-pattern, and GoF authors said they would remove it, if only they could go back in time.

But we can definitely hack our way around this using Python **Class**


## Design

Let's create a snippet of text as an example:


```python
sample = """<h1>Title Goes Here</h1>
<b>Bolded Text</b>
<i>Italicized Text</i>
<img src="this should all be gone"/>
<a href="this will be gone, too">But this will still be here!</a>
I run. He ran. She is running. Will they stop running?
I talked. She was talking. They talked to them about running. Who ran to the talking runner?
[Some text we don't want to keep is in here]
¡Sebastián, Nicolás, Alejandro and Jéronimo are going to the store tomorrow morning!
something... is! wrong() with.,; this :: sentence.
I can't do this anymore. I didn't know them. Why couldn't you have dinner at the restaurant?
My favorite movie franchises, in order: Indiana Jones; Marvel Cinematic Universe; Star Wars; Back to the Future; Harry Potter.
Don't do it.... Just don't. Billy! I know what you're doing. This is a great little house you've got here.
[This is some other unwanted text]
John: "Well, well, well."
James: "There, there. There, there."
&nbsp;&nbsp;
There are a lot of reasons not to do this. There are 101 reasons not to do it. 1000000 reasons, actually.
I have to go get 2 tutus from 2 different stores, too.
22    45   1067   445
{{Here is some stuff inside of double curly braces.}}
{Here is more stuff in single curly braces.}
[DELETE]
</body>
</html>"""
```

Say you want to strip some html characters and use regular expressions to remove open and close double brackets and anything in between them:


```python
import re, string, unicodedata
from bs4 import BeautifulSoup

def strip_html(text):
    soup = BeautifulSoup(text, "html.parser")
    return soup.get_text()

def remove_between_square_brackets(text):
    return re.sub('\[[^]]*\]', '', text)

def denoise_text(text):
    text = strip_html(text)
    text = remove_between_square_brackets(text)
    return text

sample = denoise_text(sample)
print(sample)
```
```text
    Title Goes Here
    Bolded Text
    Italicized Text
    
    But this will still be here!
    I run. He ran. She is running. Will they stop running?
    I talked. She was talking. They talked to them about running. Who ran to the talking runner?
    
    ¡Sebastián, Nicolás, Alejandro and Jéronimo are going to the store tomorrow morning!
    something... is! wrong() with.,; this :: sentence.
    I can't do this anymore. I didn't know them. Why couldn't you have dinner at the restaurant?
    My favorite movie franchises, in order: Indiana Jones; Marvel Cinematic Universe; Star Wars; Back to the Future; Harry Potter.
    Don't do it.... Just don't. Billy! I know what you're doing. This is a great little house you've got here.
    
    John: "Well, well, well."
    James: "There, there. There, there."
    
    There are a lot of reasons not to do this. There are 101 reasons not to do it. 1000000 reasons, actually.
    I have to go get 2 tutus from 2 different stores, too.
    22    45   1067   445
    {{Here is some stuff inside of double curly braces.}}
     {Here is more stuff in single curly braces.}
```   
    
This perfectly demonstrates our normal workflow:
1. Define couple of functions like `strip_html()`, `remove_punctuation()` ...
2. Run them one by one or define another "master function" to run them all like above example
3. Found that we need add more functions or change the order of the function runs
4. Make changes to the "master function" by copy and paste, then re-define it
5. Run new "master function" on the text
6. Rinse and repeat

It is not very flexiable and easy to maintain, isn't it?

## Solution

If we put those functions in a Class and let the function return `self`, we can use the dot notation `.` to chain them together, at any order!


```python
class cleantext():
    
    def __init__(self, text = None):
        self.text = text
        
    def strip_html(self):
        soup = BeautifulSoup(self.text, "html.parser")
        self.text = soup.get_text()
        return self

    def remove_between_square_brackets(self):
        self.text = re.sub('\[[^]]*\]', '', self.text)
        return self

    def remove_numbers(self):
        self.text = re.sub('[-+]?[0-9]+', '', self.text)
        return self
    
    def do_all(self, text):
        
        self.text = text
        self = self.strip_html()
        self = self.remove_numbers()
       
        return self.words


```


```python
ct = cleantext(sample)
```


```python
print(ct.strip_html().remove_between_square_brackets().remove_numbers().text)
```

```text
    Title Goes Here
    Bolded Text
    Italicized Text
    
    But this will still be here!
    I run. He ran. She is running. Will they stop running?
    I talked. She was talking. They talked to them about running. Who ran to the talking runner?
    
    ¡Sebastián, Nicolás, Alejandro and Jéronimo are going to the store tomorrow morning!
    something... is! wrong() with.,; this :: sentence.
    I can't do this anymore. I didn't know them. Why couldn't you have dinner at the restaurant?
    My favorite movie franchises, in order: Indiana Jones; Marvel Cinematic Universe; Star Wars; Back to the Future; Harry Potter.
    Don't do it.... Just don't. Billy! I know what you're doing. This is a great little house you've got here.
    
    John: "Well, well, well."
    James: "There, there. There, there."
    
    There are a lot of reasons not to do this. There are  reasons not to do it.  reasons, actually.
    I have to go get  tutus from  different stores, too.
              
    {{Here is some stuff inside of double curly braces.}}
    {Here is more stuff in single curly braces.}
```   
    
    
This makes our code readable and easy to manipulate. We can read out our code too --- just read the dot `.` as "then" in your mind : 
> sample text **then** strip html **then** remove between square brackets **then** remove numbers"


## Full implementation

So my full definition of the class looks like this: (example and many functions are from KDNugget)

```python
import re, string, unicodedata
import nltk
import contractions
import inflect
from bs4 import BeautifulSoup
from nltk import word_tokenize, sent_tokenize
from nltk.corpus import stopwords
from nltk.stem import LancasterStemmer, WordNetLemmatizer

class cleantext():
    
    def __init__(self, text = "test"):
        self.text = text
        
    def strip_html(self):
        soup = BeautifulSoup(self.text, "html.parser")
        self.text = soup.get_text()
        return self

    def remove_between_square_brackets(self):
        self.text = re.sub('\[[^]]*\]', '', self.text)
        return self

    def remove_numbers(self):
        self.text = re.sub('[-+]?[0-9]+', '', self.text)
        return self

    def replace_contractions(self):
        """Replace contractions in string of text"""
        self.text = contractions.fix(self.text)
        return self
    
    def get_words(self):
        self.words = nltk.word_tokenize(self.text)
        return self

    def remove_non_ascii(self):
        """Remove non-ASCII characters from list of tokenized words"""
        new_words = []
        for word in self.words:
            new_word = unicodedata.normalize('NFKD', word).encode('ascii', 'ignore').decode('utf-8', 'ignore')
            new_words.append(new_word)
        self.words = new_words
        return self

    def to_lowercase(self):
        """Convert all characters to lowercase from list of tokenized words"""
        new_words = []
        for word in self.words:
            new_word = word.lower()
            new_words.append(new_word)
        self.words = new_words
        return self

    def remove_punctuation(self):
        """Remove punctuation from list of tokenized words"""
        new_words = []
        for word in self.words:
            new_word = re.sub(r'[^\w\s]', '', word)
            if new_word != '':
                new_words.append(new_word)
        self.words = new_words
        return self

    def replace_numbers(self):
        """Replace all interger occurrences in list of tokenized words with textual representation"""
        p = inflect.engine()
        new_words = []
        for word in self.words:
            if word.isdigit():
                new_word = p.number_to_words(word)
                new_words.append(new_word)
            else:
                new_words.append(word)
        self.words = new_words
        return self

    def remove_stopwords(self):
        """Remove stop words from list of tokenized words"""
        new_words = []
        for word in self.words:
            if word not in stopwords.words('english'):
                new_words.append(word)
        self.words = new_words
        return self

    def stem_words(self):
        """Stem words in list of tokenized words"""
        stemmer = LancasterStemmer()
        stems = []
        for word in self.words:
            stem = stemmer.stem(word)
            stems.append(stem)
        self.words = stems
        return self

    def lemmatize_verbs(self):
        """Lemmatize verbs in list of tokenized words"""
        lemmatizer = WordNetLemmatizer()
        lemmas = []
        for word in self.words:
            lemma = lemmatizer.lemmatize(word, pos='v')
            lemmas.append(lemma)
        self.words = lemmas
        return self
    
    def join_words(self):
        self.words = ' '.join(self.words)
        return self
    
    def do_all(self, text):
        
        self.text = text
        self = self.strip_html()
        self = self.remove_numbers()
        self = self.replace_contractions()
        self = self.get_words()
        self = self.remove_punctuation()
        self = self.remove_non_ascii()
        self = self.remove_stopwords()
        self = self.stem_words()
        self = self.lemmatize_verbs()
        self = self.join_words()
        
        return self.words
```

Now we can pre-process our text easily, in a chain-like matter:


```python
ct = cleantext(sample)

ct.\
strip_html().\
remove_numbers().\
replace_contractions().\
get_words().\
remove_punctuation().\
remove_non_ascii().\
remove_stopwords().\
stem_words().\
lemmatize_verbs().\
join_words().\
words
```

```text
'titl goe her bold text it text but stil i run he run she run wil stop run i talk she talk they talk run who run talk run som text want keep sebast nicola alejandro jeronimo go stor tomorrow morn someth wrong send i anym i know why could din resta my favorit movy franch ord indian jon marvel cinem univers star war back fut harry pot just bil i know thi gre littl hous get thi unw text john wel wel wel jam ther ther ther lot reason ther reason reason act i go get tut diff stor her stuff insid doubl cur brac her stuff singl cur brac delet'
```
You can easily add or modify any step on your code, it's like a pipeline!


## Apply on Pandas DataFrame

Say right now we have a dataframe and one column contains the string we would like to process. We can easily use `apply()` 


```python
import pandas as pd
```


```python
df = pd.DataFrame({'id': range(0,100), 'docs':sample } )
```


```python
df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>docs</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>0</td>
      <td>&lt;h1&gt;Title Goes Here&lt;/h1&gt;\n&lt;b&gt;Bolded Text&lt;/b&gt;\n...</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1</td>
      <td>&lt;h1&gt;Title Goes Here&lt;/h1&gt;\n&lt;b&gt;Bolded Text&lt;/b&gt;\n...</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2</td>
      <td>&lt;h1&gt;Title Goes Here&lt;/h1&gt;\n&lt;b&gt;Bolded Text&lt;/b&gt;\n...</td>
    </tr>
    <tr>
      <th>3</th>
      <td>3</td>
      <td>&lt;h1&gt;Title Goes Here&lt;/h1&gt;\n&lt;b&gt;Bolded Text&lt;/b&gt;\n...</td>
    </tr>
    <tr>
      <th>4</th>
      <td>4</td>
      <td>&lt;h1&gt;Title Goes Here&lt;/h1&gt;\n&lt;b&gt;Bolded Text&lt;/b&gt;\n...</td>
    </tr>
  </tbody>
</table>
</div>




```python
df['cleaned_docs'] = df['docs'].apply(ct.do_all)
```


```python
df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>docs</th>
      <th>cleaned_docs</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>0</td>
      <td>&lt;h1&gt;Title Goes Here&lt;/h1&gt;\n&lt;b&gt;Bolded Text&lt;/b&gt;\n...</td>
      <td>titl goe her bold text it text but stil i run ...</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1</td>
      <td>&lt;h1&gt;Title Goes Here&lt;/h1&gt;\n&lt;b&gt;Bolded Text&lt;/b&gt;\n...</td>
      <td>titl goe her bold text it text but stil i run ...</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2</td>
      <td>&lt;h1&gt;Title Goes Here&lt;/h1&gt;\n&lt;b&gt;Bolded Text&lt;/b&gt;\n...</td>
      <td>titl goe her bold text it text but stil i run ...</td>
    </tr>
    <tr>
      <th>3</th>
      <td>3</td>
      <td>&lt;h1&gt;Title Goes Here&lt;/h1&gt;\n&lt;b&gt;Bolded Text&lt;/b&gt;\n...</td>
      <td>titl goe her bold text it text but stil i run ...</td>
    </tr>
    <tr>
      <th>4</th>
      <td>4</td>
      <td>&lt;h1&gt;Title Goes Here&lt;/h1&gt;\n&lt;b&gt;Bolded Text&lt;/b&gt;\n...</td>
      <td>titl goe her bold text it text but stil i run ...</td>
    </tr>
  </tbody>
</table>
</div>



## Parallelization using `dask`

Care for some speed up using parallelization? No problem. `dask` to the rescue. [(dask documents)](https://docs.dask.org/en/latest/)


```python
import dask.dataframe as dd
from dask.multiprocessing import get
import timeit
```

The idea here is we partition a pandas dataframe into "dask dataframe", then we can run the job parallelly by putting different partition on different workers:


```python
def dask_this(df):
    res = df['docs'].apply(ct.do_all)
    return res  
```


```python
ddata = dd.from_pandas(df, npartitions=10)
```


```python
type(ddata)
```



```text
dask.dataframe.core.DataFrame
```


Doing a benchmark on our dataframes with 100 samples: 


```python
import time
start_time = time.time()
res = ddata.map_partitions(dask_this).compute(scheduler='processes', num_workers=10)
print("--- %s seconds ---" % (time.time() - start_time))
```

    --- 1.2451355457305908 seconds ---



```python
import time
start_time = time.time()
res = df['docs'].apply(ct.do_all)
print("--- %s seconds ---" % (time.time() - start_time))
```

    --- 8.75901198387146 seconds ---


~7x speed up with 10 cores! Lets all start dasking!


![](https://media.giphy.com/media/sRKg9r2YWeCTG5JTTo/giphy.gif)

