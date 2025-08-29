## Elasticsearch ILM (Index Lifecycle Management)

Elasticsearch ILM allows us to automatically manage the indices stored on Elasticsearch. It enables us to manage previously created or to-be-created indices by assigning lifecycle policies to them.

### ILM Lifecycle Phases

- __HOT__ : This is the time when incoming data is actively written to the index and queries are performed quickly. A new index can be created when the active index reaches a certain size, a certain number of documents, or after a certain number of days. After a new index is created, old directories can be queried, but write operations are performed on the new directory.

- __WARM__ : At this stage, no new data is being written to the index, but queries can still be performed. Optionally, replicas can be reduced, shards can be shrunk, or the index can be moved to less performant hardware since no write operations are occurring.

- __COLD__ : This is when the index is completely closed to updates and becomes rarely queried. Queries are slower compared to other upper phases. Here, the index can be moved to different hardware if desired. To reduce hardware requirements, a frozen operation can be applied to the index. After the frozen operation, queries slow down because the query result is reloaded from disk to RAM.

- __DELETE__ : In the final stage, we can make arrangements to delete the index after a certain period.

### How to Create ILM Lifecycle?

We can create lifecycle policies either through the Kibana management interface or using the Kibana dev tools.<br><br> 

First, we need to create a general index pattern to easily access our data sources or indices. In this example, with the index pattern "myproject-data-*", we can manage our data sources and indices that start with myproject-data-.

```
POST .kibana/_doc/index-pattern:myproject-data
{
  "type": "index-pattern",
  "index-pattern": {
    "title": "myproject-data-*",
    "timeFieldName": "@timestamp"
  }
}
```

Next, we need to create an index template. With the JSON query below, an index template named "myproject-template" with an alias named "data-alias" and the "myproject-data-*" index pattern is created.

```
PUT /_index_template/myproject-template
{
  "index_patterns": "myproject-data-*",
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 1
    },
    "mappings": {
      "properties": {
        "timestamp": {
          "type": "date"
        },
        "message": {
          "type": "text"
        },
        "level": {
          "type": "keyword"
        }
      }
    },
   "aliases": {
    "data-alias": {}
  }
}
}
```

We need to determine an index that manages the indices. In this example, we create an index named "myproject-write-index".

```
PUT /myproject-write-index
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1
  },
  "mappings": {
    "properties": {
      "timestamp": {
        "type": "date"
      },
      "message": {
        "type": "text"
      },
      "level": {
        "type": "keyword"
      }
    }
  }
}
```

At this stage, we associate the write-index we created with the alias named data-alias.

```
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "myproject-write-index",
        "alias": "data-alias",
        "is_write_index": true
      }
    }
  ]
}
```

Now we can create a lifecycle policy. In this example, indices older than 15 days will be deleted.

```
PUT /_ilm/policy/myproject-lifecycle-policy
{
  "policy": {
    "phases": {
      "delete": {
        "min_age": "15d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

Finally, we need to associate our indices with the "myproject-data-*" pattern with the policy we created using the alias.

```
PUT /myproject-data-*/_settings
{
  "settings": {
    "index.lifecycle.name": "myproject-lifecycle-policy",
    "index.lifecycle.rollover_alias": "data-alias"
  }
}
```
