---
title: my first post
author: Oliver P. Madsen
date: '2020-11-21'
slug: my-first-post-manul
categories: ["new post"]
tags: ['tech', 'initial post']
keywords:
  - tech
  - initial post
---
This is a header!

<!--more--> 
Hello world  
I am fantastic

<!-- toc -->
I am trying to embed an image.


```r
data(mtcars)
library(ggplot2)
ggplot(mtcars, aes(x = hp, y = mpg)) + 
  geom_point() + 
  geom_smooth()
```


<!-- toc -->
# Embedding with markdown
Here I'm embedding an image using standard markdown syntax. It works as expected.

```markdown
![alt text](index_files/figure-html/test-1.png)
```

![alt text](index_files/figure-html/test-1.png)
# Embedding using figure

Here I'm embedding an image using figure. It seems to work.

```markdown
{{<figure src="index_files/figure-html/test-1.png" width="672" >}}
```

{{<figure src="index_files/figure-html/test-1.png" width="672" >}}

# Embedding using img

Here I'm embedding using img, which is standard for rmarkdown files. It does not seem to work.
```markdown
<img src="index_files/figure-html/test-1.png" width="672" />
```

<img src="index_files/figure-html/test-1.png" width="672" />



