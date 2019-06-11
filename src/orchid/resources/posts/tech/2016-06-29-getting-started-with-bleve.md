---
title: Getting started with bleve
tags: ['search']
---
I recently made a little prototype for a search server. Normally I would have used [Elasticsearch](https://www.elastic.co/products/elasticsearch) for such a case, but I wanted to write this thing in pure Go. After looking at some alternatives, like a [lucene port to Go](https://github.com/balzaczyy/golucene), which seemed to be “done” since 2015\. Lucene 4.10 was also not very appealing to me. While searching for a solution, I stumbled over [Bleve](http://www.blevesearch.com/), which is a Go-only full-text indexing and search library implementation.

As I still had [Elasticsearch](https://www.elastic.co/products/elasticsearch) in the back of my hand, so I gave it a try. I use [GB](https://getgb.io/) for my Go projects, so please do not wonder about my imports in the source snippets. GB enables something that feels to me like a real project structure and I can really recommend it (although it lacks of the support to run `go run`).

I did a lot of [Elasticsearch](https://www.elastic.co/products/elasticsearch) projects in my past, so I hoped, that the basic knowledge of how to index something would help me to get things up and running fast.

I started with the data model. I took a shop system from a friend, politely stole his article database and used it for my indexing test.

{% highlight language="Go" %}
// Article represents a single article
type Article struct {
    ArticleID int `json:"article_id"`
    Name string `json:"name"`
    OrderNumber string `json:"order_number"`
    SalesCount int `json:"sales_count"`
    Keywords []string `json:"keywords"`
    Color string `json:"color"`
    Locale string `json:"locale"`
    TranslatedName string `json:"translated_name"`
}

// Type refers to the document type in bleve
func (a *Article) Type() string {
    return "article"
}
{% endhighlight %}

Bleve is based on file indexes, which can be stored in different backends. I tried LevelDB and BoltDB, where the [latter is the faster one](https://github.com/blevesearch/bleve-bench#conclusions). Bleve makes heavy use of compile tags, where `+boltdb` is one of them.

I came up with an `indexing` package, that should contain the related sources for indexing. First step: create and open an index.

{% highlight language="Go" %}
package indexing

import (
    log "github.com/Sirupsen/logrus"
    "github.com/blevesearch/bleve"
    "os"
)

// OpenIndex returns the opened index
func OpenIndex(databasePath string) bleve.Index {
    index, err := bleve.Open(databasePath)

    if err != nil {
        log.Fatal(err)
        os.Exit(-1)
    }

    return index
}

// CreateIndex creates the initial index
func CreateIndex(databasePath string) bleve.Index {
    mapping := bleve.NewIndexMapping()
    mapping = addCustomAnalyzers(mapping)
    mapping = createArticleMapping(mapping)
    index, err := bleve.New(databasePath, mapping)
    if err != nil {
        log.Fatal(err)
        os.Exit(-1)
    }
    return index
}
{% endhighlight %}

The code points out, that there are two more functions, that support the index creation: `addCustomAnalyzers` and `createArticleMapping.`

In order make a search useful, you need to customize the process of analyzing the terms you want to index. The standard is in most cases worth nothing. In my case, I wanted to have an edge-ngram tokenizer for the `translated_name`.

{% highlight language="Go" %}
package indexing

import (
    "github.com/blevesearch/bleve"
)

func createArticleMapping(indexMapping *bleve.IndexMapping) *bleve.IndexMapping {
    articleMapping := bleve.NewDocumentMapping()

    articleIDMapping := bleve.NewNumericFieldMapping()
    articleIDMapping.IncludeInAll = false
    articleMapping.AddFieldMappingsAt("article_id", articleIDMapping)

    nameMapping := bleve.NewTextFieldMapping()
    nameMapping.IncludeInAll = false
    articleMapping.AddFieldMappingsAt("name", nameMapping)

    orderNumberMapping := bleve.NewTextFieldMapping()
    orderNumberMapping.IncludeInAll = false
    orderNumberMapping.IncludeTermVectors = false
    articleMapping.AddFieldMappingsAt("order_number", orderNumberMapping)

    salesCountMapping := bleve.NewNumericFieldMapping()
    salesCountMapping.IncludeInAll = false
    articleMapping.AddFieldMappingsAt("sales_count", salesCountMapping)

    keywordsMapping := bleve.NewTextFieldMapping()
    articleMapping.AddFieldMappingsAt("keywords", keywordsMapping)

    translatedName := bleve.NewTextFieldMapping()
    translatedName.Analyzer = "fulltext_ngram"
    articleMapping.AddFieldMappingsAt("translated_name", translatedName)

    color := bleve.NewTextFieldMapping()
    color.IncludeInAll = false
    articleMapping.AddFieldMappingsAt("color", color)

    locale := bleve.NewTextFieldMapping()
    locale.IncludeInAll = false
    locale.Analyzer = "not_analyzed"
    articleMapping.AddFieldMappingsAt("locale", locale)

    indexMapping.AddDocumentMapping("article", articleMapping)

    return indexMapping
}
{% endhighlight %}

and the custom analyzers:

{% highlight language="Go" %}
package indexing

import (
    log "github.com/Sirupsen/logrus"
    "github.com/blevesearch/bleve"
    "github.com/blevesearch/bleve/analysis/analyzers/custom_analyzer"
    "github.com/blevesearch/bleve/analysis/token_filters/edge_ngram_filter"
    "github.com/blevesearch/bleve/analysis/token_filters/lower_case_filter"
    "github.com/blevesearch/bleve/analysis/tokenizers/single_token"
    "github.com/blevesearch/bleve/analysis/tokenizers/unicode"
)

func addCustomTokenFilter(indexMapping *bleve.IndexMapping) *bleve.IndexMapping {
    err := indexMapping.AddCustomTokenFilter("bigram_tokenfilter", map[string]interface{}{
        "type": edge_ngram_filter.Name,
        "side": edge_ngram_filter.FRONT,
        "min":  3.0,
        "max":  25.0,
    })

    if err != nil {
        log.Fatal(err)
    }

    return indexMapping
}

func addCustomAnalyzers(indexMapping *bleve.IndexMapping) *bleve.IndexMapping {
    indexMapping = addCustomTokenFilter(indexMapping)

    err := indexMapping.AddCustomAnalyzer("not_analyzed", map[string]interface{}{
        "type":      custom_analyzer.Name,
        "tokenizer": single_token.Name,
    })

    if err != nil {
        log.Fatal(err)
    }

    err = indexMapping.AddCustomAnalyzer("fulltext_ngram", map[string]interface{}{
        "type":      custom_analyzer.Name,
        "tokenizer": unicode.Name,
        "token_filters": []string{
            lower_case_filter.Name,
            "bigram_tokenfilter",
        },
    })

    if err != nil {
        log.Fatal(err)
    }

    return indexMapping
}
{% endhighlight %}

Setting up the indexing process was the least effort, compared to understanding how the mapping works in Bleve. You need to read much of their source code, the wiki gives you only a very high level overview.

Finally, here is the code I’ve used to index the article data, bringing it all together. You can assume, that the slice of articles was loaded from a MySQL database and mapped to the article struct you’ve seen in above’s example.

{% highlight language="Go" %}
// Execute the import step
func (istep *ImportFromMySQL) Execute(barContainer *multibar.BarContainer) {
    articles := istep.getArticles()
    idxProgress := barContainer.MakeBar(len(articles), "Indexing")
    go barContainer.Listen()
    for k, v := range articles {
        log.WithFields(log.Fields{
            "OrderID":        v.OrderNumber,
            "Color":          v.Color,
            "TranslatedName": v.TranslatedName,
            "Locale":         v.Locale,
        }).Debug("Indexing Article")
        id := fmt.Sprintf("%s-%s", string(v.OrderNumber), v.Locale)
        istep.index.Index(id, v)
        idxProgress(k + 1)
    }
    idxProgress(len(articles))
}
{% endhighlight %}

At the end, I’ve switched it all to Elasticsearch, due to performance reasons. A search request took in the best case 60ms and in the worst case about 300ms. Compared to Elasticsearch, even with taking the HTTP overhead into consideration, I get results below 10ms on my local 2015 MacBook Pro. My personal decision to go back to Elasticsearch was truly biased by the availability of the much deeper knowledge of the Elasticsearch internals. Bleve seems to my very appealing, as it can be a no-other-server-needed way of building a medium complex search service. I will follow its progress and maybe retest it some day. I am thankful anyway that somebody made the effort of creating such an educated library in the Go world. Keep up the good work!

I hope these code bits help you to get started with Bleve, it was a lot of fun for me.
