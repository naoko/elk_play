>> **Elasticsearch** is a highly scalable open-source full-text search and analytic engine. It allows you to store, search, and analyze big volumes of data quickly and in near real time. It is generally used as the underlying engine/technology that powers applications that have complex search features and requirements.

###basic concepts

**newar realtime (nrt)**: Elasticsearch is a near real time search platform. What this means is there is a slight latency (normally one second) from the time you index a document until the time it becomes searchable.

**cluster**: A cluster is a collection of one or more nodes (servers) that together holds your entire data and provides federated indexing and search capabilities across all nodes. A cluster is identified by a unique name which by default is "elasticsearch". This name is important because a node can only be part of a cluster if the node is set up to join the cluster by its name.

**node**: A node is a single server that is part of your cluster, stores your data, and participates in the cluster’s indexing and search capabilities. Just like a cluster, a node is identified by a name which by default is a random Marvel character name that is assigned to the node at startup.

**index**: An index is a collection of documents that have somewhat similar characteristics. In a single cluster, you can define as many indexes as you want.

**type**: Within an index, you can define one or more types. A type is a logical category/partition of your index whose semantics is completely up to you. In general, a type is defined for documents that have a set of common fields. 

**document**: A document is a basic unit of information that can be indexed and expressed in JSON.

**shards & replicas**: An index can potentially store a large amount of data that can exceed the hardware limits of a single node. To solve this problem, Elasticsearch provides the ability to subdivide your index into multiple pieces called shards.  When you create an index, you can simply define the number of shards that you want. Each shard is in itself a fully-functional and independent "index" that can be hosted on any node in the cluster.Sharding is important for two primary reasons:

* It allows you to horizontally split/scale your content volume
* It allows you distribute and parallelize operations across shards (potentially on multiple nodes) thus increasing performance/throughput

The mechanics of how a shard is distributed and also how its documents are aggregated back into search requests are completely managed by Elasticsearch and is transparent to you as the user.

In a network/cloud environment where failures can be expected anytime, it is very useful and highly recommended to have a failover mechanism in case a shard/node somehow goes offline or disappears for whatever reason. To this end, Elasticsearch allows you to make one or more copies of your index’s shards into what are called replica shards, or replicas for short.

Replication is important for two primary reasons:

* It provides high availability in case a shard/node fails. For this reason, it is important to note that a replica shard is never allocated on the same node as the original/primary shard that it was copied from.
* It allows you to scale out your search volume/throughput since searches can be executed on all replicas in parallel.

To summarize, each index can be split into multiple shards. An index can also be replicated zero (meaning no replicas) or more times. Once replicated, each index will have primary shards (the original shards that were replicated from) and replica shards (the copies of the primary shards). The number of shards and replicas can be defined per index at the time the index is created. After the index is created, you may change the number of replicas dynamically anytime but you cannot change the number shards after-the-fact.

By default, each index in Elasticsearch is allocated 5 primary shards and 1 replica which means that if you have at least two nodes in your cluster, your index will have 5 primary shards and another 5 replica shards (1 complete replica) for a total of 10 shards per index.

###installation
**source: https://github.com/Traackr/ansible-elasticsearch**

launch server with vagrant and provisiong with ansible

```
cd your_elk_project
virtualenv venv
source venv/bin/activate
pip install -r requirements.txt

vagrant plugin install vagrant-vbguest

cd ansible-elasticsearch/
vagrant up
vagrant provision
```

##getting started

ref: http://bract.co/x5QW

###create an index
```
server=http://192.168.111.10:9200

# green means good
$ curl ${server}/_cat/health?v
epoch      timestamp cluster       status node.total node.data shards pri relo init unassign
1414896976 19:56:16  elasticsearch green           1         1      0   0    0    0        0

# get a list of nodes in our cluster 
$ curl ${server}/_cat/nodes?v
precise64 127.0.1.1            6          10 0.39 d         *      Flambe

# list indexes
$ curl ${server}/_cat/indices?v
health index pri rep docs.count docs.deleted store.size pri.store.size
# which simply means we have no indexes yet in the cluster.

# create an index
# command creates the index named "customer" using the PUT verb. 
# We simply append pretty to the end of the call to tell it to pretty-print the JSON response 
curl -XPUT ${server}/customer?pretty

# we now have 1 index named customer and it has 5 primary shards and 1 replica (the defaults) and it contains 0 documents in it.
# You might also notice that the customer index has a yellow health tagged to it. 
# yellow means that some replicas are not (yet) allocated. The reason this happens for this index is because Elasticsearch by default created one replica for this index. Since we only have one node running at the moment, that one replica cannot yet be allocated
curl ${server}/_cat/indices?v
```

###index and query a document
```
server=http://192.168.111.10:9200

# Let’s index a simple customer document into the customer index, "external" type, with an ID of 1 as follows
# our JSON document: { "name": "John Doe" }

curl -XPUT $server/customer/external/1?pretty -d '
{
  "name": "John Doe"
}'
```
It is important to note that Elasticsearch does not require you to explicitly create an index first before you can index documents into it. 

```
# retrieve that document
curl -XGET $server/customer/external/1?pretty
```
this should return
```
{
  "_index" : "customer",
  "_type" : "external",
  "_id" : "1",
  "_version" : 1,
  "found" : true,  # we found document
  "_source":  # returns the full JSON document
{
  "name": "John Doe"
}
}
```

###delete an index
```
curl -XDELETE $server/customer?pretty
curl $server/_cat/indices?v
```

This REST access pattern is pervasive 
```
curl -<REST Verb> <Node>:<Port>/<Index>/<Type>/<ID>
```

###updating your data
```
# replacing existing document
curl -XPUT $server/customer/external/1?pretty -d '
{
  "name": "Jane Doe",
  "age": 20
}'

# insert data without specifying id
curl -XPOST $server:9200/customer/external?pretty -d '
{
  "name": "Jane Doe"
}'

# update document
curl -XPOST $server/customer/external/1/_update?pretty -d '
{
  "doc": { "name": "Jane Doe" }
}'

# update with scripts - increment age by 5
# ctx._source refers to the current source document that is about to be updated
curl -XPOST $$server/customer/external/1/_update?pretty -d '
{
  "script" : "ctx._source.age += 5"
}'
```
###deleting your data
```
# delete by id
curl -XDELETE $server/customer/external/2?pretty

# delete all customers whose names contain "John"
curl -XDELETE $server/customer/external/_query?pretty -d '
{
  "query": { "match": { "name": "John" } }
}'
```
###batch processing
```
# indexes two documents (ID 1 - John Doe and ID 2 - Jane Doe) in one bulk operation
curl -XPOST $server/customer/external/_bulk?pretty -d '
{"index":{"_id":"1"}}
{"name": "John Doe" }
{"index":{"_id":"2"}}
{"name": "Jane Doe" }
'

# updates the first document (ID of 1) and then deletes the second document (ID of 2) in one bulk operation
curl -XPOST $server/customer/external/_bulk?pretty -d '
{"update":{"_id":"1"}}
{"doc": { "name": "John Doe becomes Jane Doe" } }
{"delete":{"_id":"2"}}
'
```
The bulk API executes all the actions sequentially and in order. If a single action fails for whatever reason, it will continue to process the remainder of the actions after it. When the bulk API returns, it will provide a status for each action (in the same order it was sent in) so that you can check if a specific action failed or not.

###exploring your data
####the search api
```
# load sample data
curl -XPOST $server/bank/account/_bulk?pretty --data-binary @accounts.json
curl $server/_cat/indices?v
```
response
```
curl $server/_cat/indices?v
health index pri rep docs.count docs.deleted store.size pri.store.size
yellow bank    5   1       1000            0    424.4kb        424.4kb
```
Which means that we just successfully bulk indexed 1000 documents into the bank index (under the account type).

####introducing the query language
```
# search all documents 
curl $server/bank/_search?q=*&pretty

# or

curl -XPOST $server/bank/_search?pretty -d '
{
  "query": { "match_all": {} }
}'
```

* **took** – time in milliseconds for Elasticsearch to execute the search
* **timed_out** – tells us if the search timed out or not
* **_shards** – tells us how many shards were searched, as well as a count of the successful/failed searched shards
* **hits** – search results
* **hits.total** – total number of documents matching our search criteria
* **hits.hits** – actual array of search results (defaults to first 10 documents)
* **_score** and **max_score** - The score is a numeric value that is a relative measure of how well the document matches the search query that we specified. The higher the score, the more relevant the document is.

```
# list all limit 1 (default is 10)
curl -XPOST $server/bank/_search?pretty -d '
{
  "query": { "match_all": {} },
  "size": 1
}'

# list 11 to 20
curl -XPOST $server/bank/_search?pretty -d '
{
  "query": { "match_all": {} },
  "from": 10,
  "size": 10
}'

# order by
curl -XPOST $server/bank/_search?pretty -d '
{
  "query": { "match_all": {} },
  "sort": { "balance": { "order": "desc" } }
}'

# specify fields to return
curl -XPOST $server/bank/_search?pretty -d '
{
  "query": { "match_all": {} },
  "_source": ["account_number", "balance"]
}'

# returns account numbered 20
curl -XPOST $server/bank/_search?pretty -d '
{
  "query": { "match": { "account_number": 20 } }
}'

# returns all accounts containing the term "mill" in the address
curl -XPOST $server/bank/_search?pretty -d '
{
  "query": { "match": { "address": "mill" } }
}'

# returns all accounts containing the term "mill" or "lane" in the address
curl -XPOST $server/bank/_search?pretty -d '
{
  "query": { "match": { "address": "mill lane" } }
}'

# match (match_phrase) that returns all accounts containing the phrase "mill lane" in the address
curl -XPOST $server/bank/_search?pretty -d '
{
  "query": { "match_phrase": { "address": "mill lane" } }
}'

# This example composes two match queries and returns all accounts containing "mill" and "lane" in the address
# the bool must clause specifies all the queries that must be true for a document to be considered a match.
curl -XPOST $server/bank/_search?pretty -d '
{
  "query": {
    "bool": {
      "must": [
        { "match": { "address": "mill" } },
        { "match": { "address": "lane" } }
      ]
    }
  }
}'

# this example composes two match queries and returns all accounts containing "mill" or "lane" in the address
# bool should clause specifies a list of queries either of which must be true for a document to be considered a match
curl -XPOST $server/bank/_search?pretty -d '
{
  "query": {
    "bool": {
      "should": [
        { "match": { "address": "mill" } },
        { "match": { "address": "lane" } }
      ]
    }
  }
}'

# This example composes two match queries and returns all accounts that contain neither "mill" nor "lane" in the address
# bool must_not clause specifies a list of queries none of which must be true
curl -XPOST $server/bank/_search?pretty -d '
{
  "query": {
    "bool": {
      "must_not": [
        { "match": { "address": "mill" } },
        { "match": { "address": "lane" } }
      ]
    }
  }
}'

# We can combine must, should, and must_not clauses simultaneously inside a bool query. Furthermore, we can compose bool queries inside any of these bool clauses to mimic any complex multi-level boolean logic.
# This example returns all accounts of anybody who is 40 years old but don’t live in ID(aho)
curl -XPOST $server/bank/_search?pretty -d '
{
  "query": {
    "bool": {
      "must": [
        { "match": { "age": "40" } }
      ],
      "must_not": [
        { "match": { "state": "ID" } }
      ]
    }
  }
}'
```

####executing filters

* Filters do not score so they are faster to execute than queries
* Filters can be cached in memory allowing repeated search executions to be significantly faster than queries

To understand filters, let’s first introduce the filtered query, which allows you to combine a query (like match_all, match, bool, etc.) together with a filter. As an example, let’s introduce the range filter, which allows us to filter documents by a range of values. This is generally used for numeric or date filtering.

This example uses a filtered query to return all accounts with balances between 20000 and 30000, inclusive. In other words, we want to find accounts with a balance that is greater than or equal to 20000 and less than or equal to 30000.
```
curl -XPOST $server/bank/_search?pretty -d '
{
  "query": {
    "filtered": {
      "query": { "match_all": {} },
      "filter": {
        "range": {
          "balance": {
            "gte": 20000,
            "lte": 30000
          }
        }
      }
    }
  }
}'
```

####executing aggregations
aggregations is by roughly equating it to the SQL GROUP BY and the SQL aggregate functions.

this example groups all the accounts by state, and then returns the top 10 (default) states sorted by count descending (also default)
```
curl -XPOST $server/bank/_search?pretty -d '
{
  "size": 0,  # not show search hits
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state"
      }
    }
  }
}'

# calculates the average account balance by state (again only for the top 10 states sorted by count in descending order)
# You can nest aggregations inside aggregations arbitrarily to extract pivoted summarizations that you require from your data.
curl -XPOST $server/bank/_search?pretty -d '
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state"
      },
      "aggs": {
        "average_balance": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  }
}'

# This example demonstrates how we can group by age brackets (ages 20-29, 30-29, and 40-49), then by gender, and then finally get the average account balance, per age bracket, per gender
curl -XPOST $server/bank/_search?pretty -d '
{
  "size": 0,
  "aggs": {
    "group_by_age": {
      "range": {
        "field": "age",
        "ranges": [
          {
            "from": 20,
            "to": 30
          },
          {
            "from": 30,
            "to": 40
          },
          {
            "from": 40,
            "to": 50
          }
        ]
      },
      "aggs": {
        "group_by_gender": {
          "terms": {
            "field": "gender"
          },
          "aggs": {
            "average_balance": {
              "avg": {
                "field": "balance"
              }
            }
          }
        }
      }
    }
  }
}'
```



