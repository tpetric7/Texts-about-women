---
title: "Melania Trump"
author: "Teodor Petrič"
date: "2020-11-20"
output: html_document
---

```{r}
knitr::opts_chunk$set(
	echo = TRUE,
	message = FALSE,
	warning = FALSE
)

```


```{r message=FALSE}
library(quanteda)
opt <- quanteda_options()
# Parallel computing of 8 processor cores
opt$threads = 8
library(quanteda.textstats)
library(quanteda.textplots)
library(quanteda.textmodels)
library(readtext)

library(tidyverse)
require(lubridate)
library(ggplot2)

library(readxl)
library(writexl)

library(udpipe)
library(lattice)
library(igraph)
library(ggraph)
library(textrank)
require("RColorBrewer")
library(wordcloud)

```

# Preberi besedila

```{r}
library(readxl)
bild_melania <- read_xlsx("texte_frauen/Bild_Reicht_Melania_Scheidung_ein.xlsx")
focus_melania <- read_xlsx("texte_frauen/Focus_Melania_will_aufgeben.xlsx")
spiegel_melania <- read_xlsx("texte_frauen/spiegel_ohne-first-lady.xlsx")
spiegel_jill <- read_xlsx("texte_frauen/spiegel_Jill Biden_loyal_konsequent_tough.xlsx")

melania <- rbind(bild_melania[, c(2,4,5,6)], focus_melania[, c(3,4,5,6)], 
                 spiegel_melania[, c(4,5,6,7)])
melania$source <- c("bild", "focus", "spiegel")

```

# Ustvari korpus (jezikovno gradivo)

```{r}
corp_mel <- corpus(melania, text_field = "body_text")

```

# Povzetek jezikovnega gradiva

```{r}
mel_stats <- summary(corp_mel)

```

# Pridobi besedne oblike in odstrani nezaželene znake in besedne oblike

```{r}
stoplist_de <- c(stopwords("german"), "dass")
toks_mel <- tokens(corp_mel, remove_punct = T, remove_numbers = T, remove_separators = T, 
                   remove_symbols = T, remove_url = T)
toks_mel_st <- tokens_select(toks_mel, pattern = stoplist_de, selection = "remove")
toks_mel_st

```

# Ustvari matriko besedil in značilnosti (DFM)

```{r}
dfm_mel <- dfm(toks_mel, remove = stoplist_de)
dfm_mel
topfeatures(dfm_mel[1,])
topfeatures(dfm_mel[2,])
topfeatures(dfm_mel[3,])

```

```{r}
library(ggplot2)
# frequency plot
dfm_mel %>% 
  textstat_frequency(n = 20) %>% 
  ggplot(aes(x = reorder(feature, frequency), y = frequency)) +
  geom_col(fill="steelblue") +
  coord_flip() +
  labs(x = NULL, y = "Frequency") +
  theme_minimal()

```


# Besedni oblaček

```{r}
set.seed(1320)
library(wordcloud2)
topfeat <- as.data.frame(topfeatures(dfm_mel,100))
topfeat <- rownames_to_column(topfeat, var = "word")
wordcloud2(topfeat)

```

```{r fig.width=10, fig.height=8}
set.seed(132)
textplot_wordcloud(dfm_mel, max_words = 150, color = c("black", "blue", "darkgreen", "magenta", "purple", "red"), min_count = 2, adjust = 0.03, rotation = 0.3, comparison = T)

```

# Pridobi matriko sopojavljanja besednih oblik (FCM) 

```{r}
dfm_tags <- dfm_select(dfm_mel, pattern = (c("melania*", "donald*", "jared*", "barron*", "wahl*", "ehevertrag*", "niederlage*", "*betrug*", "joe", "biden*")))
toptag <- names(topfeatures(dfm_tags, 50))
head(toptag)

```


```{r}
# Construct feature-cooccurrence matrix (fcm) of selected words
fcm_mel <- fcm(dfm_mel)
head(fcm_mel)
top_fcm <- fcm_select(fcm_mel, pattern = toptag)
textplot_network(top_fcm, min_freq = 0.1, edge_alpha = 0.8, edge_size = 5)

# 1. Open jpeg file
jpeg("pictures/melania_fcm1.jpg", 
     width = 840, height = 535)
# 2. Create the plot
textplot_network(top_fcm, min_freq = 0.1, edge_alpha = 0.8, edge_size = 5)
# 3. Close the file
dev.off()

# Open a pdf file
pdf("pictures/melania_fcm1.pdf") 
# 2. Create a plot
textplot_network(top_fcm, min_freq = 0.1, edge_alpha = 0.8, edge_size = 5)
# Close the pdf file
dev.off() 

```


# Analiza besednih oblik z jezikovnim modelom UD

```{r}
library(udpipe)
udmodel_german <- udpipe_download_model(language = "german")
# udmodel_german <- udpipe_load_model(udmodel_german$file_model)
file_model = "german-gsd-ud-2.5-191206.udpipe"
udmodel_german <- udpipe_load_model(file_model)

x <- udpipe_annotate(udmodel_german, x = texts(corp_mel[1:3]), trace = TRUE)
x <- as.data.frame(x)
str(x)

write.table(as_conllu(x), file = "texte_frauen/Melania_ud.conllu", sep = "\t", quote = F, row.names = F)

```

# Število pojavnic (token frequency)

```{r}
# Compute token frequencies 
tokens_freq <- table(x$token)
head(tokens_freq, 20)

```

```{r}
#Top 20
sort(tokens_freq, decreasing = TRUE)[1:20]

```

```{r}
#The same for lemmas
lemmas_freq <- table(x$lemma)
sort(lemmas_freq, decreasing = TRUE)[1:20]

```

# Besedni oblaček

```{r}
set.seed(1322)
library(wordcloud2)
lems <- as.data.frame(lemmas_freq)
colnames(lems) <- c("word", "freq")
lems$word <- as.character(lems$word)
lems$freq <- as.numeric(lems$freq)
wordcloud2(lems, shape = "star")

```


```{r}
xlems <- tokens(x$lemma, remove_numbers = T, remove_punct = T, remove_symbols = T, remove_url = T, remove_separators = T)
dfm_xlems <- dfm(xlems, remove = stoplist_de)

set.seed(1322)
topfeat <- as.data.frame(topfeatures(dfm_xlems,200))
topfeat <- rownames_to_column(topfeat, var = "word")
wordcloud2(topfeat, shape = "circle")

```

