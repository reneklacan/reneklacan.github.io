---
layout:     post
title:      Elasticsearch tips you might not know
date:       2015-02-07 00:00:07
summary:
categories: elasticsearch
---

### 1. Routing

With routing you are able to store set of documents on the same shard and then execute the query on a single shard. The simple example where this feature can come handy is application containing data for lot of clients where most queries are filtering data just to particular client. Then by querying single shard you are able to avoid overhead caused by joining results from multiple shards (which grows with number of nodes).

You can even set up routing by field’s value in mapping like this

{% highlight json %}
{
  "book": {
    "_routing": {
      "required": true,
      "path": "client_id"
    }
  }
}
{% endhighlight %}

Where path’s value can point to any non_analyzed field from the mapping. In this case queries on client_id would be done on one shard.

You can also have field dedicated just to routing, for example if path’s value would be _routing then you can do


{% highlight bash %}
curl -XPOST http://localhost:9200 -d '{ “name”: “foobar”, “_routing”: “abc” }'
{% endhighlight %}

For more info: [http://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-routing-field.html](http://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-routing-field.html)

### 2. Logging slow queries

After setting following configuration queries taking more than threshold will be logged

{% highlight yaml %}
index.search.slowlog.threshold.query.info: 400ms
index.search.slowlog.threshold.query.debug: 200ms
index.search.slowlog.threshold.fetch.info: 800ms
index.search.slowlog.threshold.fetch.debug: 400ms
http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/index-modules-slowlog.html
{% endhighlight %}

### 3. Aliases + Reindexing

Sooner or later you will probably run into problem that you will need to change mapping and that also means that you have to reindex all your data. You can make your life simpler and more beautiful by giving your indices names like “books\_20150115” and then creating alias “books”. Yours application will use “books” index and later on you will be able to reindex your data into to “books\_20150303” and then make “books” alias to point to a new index with just one command. It’s very production friendly.

For more info: [http://www.elastic.co/guide/en/elasticsearch/reference/current/indices-aliases.html](http://www.elastic.co/guide/en/elasticsearch/reference/current/indices-aliases.html)

### 4. Warmer API

You can use Warmer API to add queries that will be run and cached after you start ElasticSearch and before you will be able to search on index. By having right queries in Warming API you can drastically reduce request times of first queries after restart of your ElasticSearch server.

For more info: [http://www.elastic.co/guide/en/elasticsearch/reference/current/indices-warmers.html](http://www.elastic.co/guide/en/elasticsearch/reference/current/indices-warmers.html)

### 5. Posting formats

This features gives you power to control how index files are written. This can be very useful when you know your goals. For example one of the advantages of Suggest API is that it’s using in-memory FST structures to store data so it’s blazingly fast. But you can actually achieve this on any field by setting “posting_format” to “memory” in your mapping


{% highlight json %}
{
  "properties": {
    "name": {
      "type": "string",
      "posting_format": "memory"
    }
  }
}
{% endhighlight %}

There are lot of other types available that can be handy in some use cases. You just have to know what you are doing as you can make big impact on resources consumption or you can even slow down your queries.

For more info: [http://www.elastic.co/guide/en/elasticsearch/reference/0.90/index-modules-codec.html](http://www.elastic.co/guide/en/elasticsearch/reference/0.90/index-modules-codec.html)

### 6. Slop

You probably know this one but in case you don’t this parameter allows us to define how many words can be in between to have a match


{% highlight json %}
{
  "query_string": {
    "query": “super movie”,
    "phrase_slop": 0
  }
}
{% endhighlight %}

Without using slop parameter this query would match “super boring movie” but when slop parameter equals to zero it can only match word super followed by movie. You can set slop to any number.
