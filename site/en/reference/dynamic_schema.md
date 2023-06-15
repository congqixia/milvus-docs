---
id: dynamic_schema.md
related_key: dynamic_schema
summary: Dynamic schema in Milvus.
---

# Dynamic Schema

Schema design is crucial for Milvus data processing. Before inserting entities into a Milvus collection, clarify the schema design and ensure that all data entities inserted afterward match the schema. However, this limits Milvus collections, making them similar to tables in relational databases. 

Dynamic schema enables users to insert entities with new fields into a Milvus collection without modifying the existing schema. This means that users can insert data without knowing the full schema of a collection and can include fields that are not yet defined.

## Create collection with dynamic schema enabled

To create a collection using a dynamic data model, set `enable_dynamic_field` to `True` when defining the data model. Afterward, all undefined fields and their values in the data entities inserted afterward will be stored in a magic JSON field named `$meta` as key-value pairs. We prefer to use the term "dynamic fields" to refer to these key-value pairs.

Notice that the `$meta` field does not bring any changes to the way you use Milvus. You can ask Milvus to output dynamic fields in search/query results and include them in search and query filter expressions just as they are already defined in the collection schema.

```python
from pymilvus import connections, Collection, FieldSchema, CollectionSchema, DataType, utility

# 1. define fields
fields = [
    FieldSchema(name="id", dtype=DataType.INT64, is_primary=True, auto_id=True, max_length=100),
    FieldSchema(name="title", dtype=DataType.VARCHAR, max_length=512),
    FieldSchema(name="title_vector", dtype=DataType.FLOAT_VECTOR, dim=768)
]
# 2. enable dynamic schema in schema definition
schema = CollectionSchema(
        fields, 
        "The schema for a medium news collection", 
        enable_dynamic_field=True
)

# 3. reference the schema in a collection
collection = Collection("medium_articles_with_dynamic", schema)

# 4. index the vector field and load the collection
index_params = {
    "index_type": "AUTOINDEX",
    "metric_type": "L2",
    "params": {}
}

collection.create_index(
  field_name="title_vector", 
  index_params=index_params
)

# 5. load the collection
collection.load()
```

## Insert dynamic data

Once the collection is created, you can start inserting data, including the dynamic data into the collection.

### Prepare data

Suppose we have a dataset with each row being similar to the following:

```json
{
   'title': 'The Reported Mortality Rate of Coronavirus Is Not Important',
   'title_vector': [0.041732933, 0.013779674, -0.027564144, ..., 0.030096486],
   'link': 'https://medium.com/swlh/the-reported-mortality-rate-of-coronavirus-is-not-important-369989c8d912',
   'reading_time': 13,
   'publication': 'The Startup',
   'claps': 1100,
   'responses': 18
}
```

## Insert data

You can insert this dataset into the collection we have just created.

```python
# 6. insert data
collection.insert(data_rows)
collection.flush()

print("Entity counts: ", collection.num_entities)

# Output
# Entity counts:  5979
```

## Search with dynamic fields

If you have created a collection with dynamic field enabled, and inserted data with dynamic fields into, index, and load the collection, you can use dynamic fields in the filter expression of a search or a query as follows:

```python
# Use the vector field of the first entity as the query vector.
result = collection.search(
    data=[data_rows[0]['title_vector']],
    anns_field="title_vector",
    param={"metric_type": "L2", "params": {"nprobe": 10}},
    limit=3,
    expr='$meta["claps"] > 30 and reading_time < 10',
    output_fields=["title", "reading_time", "claps"],
)

for hits in result:
    print("Matched IDs: ", hits.ids)
    print("Distance to the query vector: ", hits.distances)
    print("Matched articles: ")
    for hit in hits:
        print(
            "Title: ", 
            hit.entity.get("title"), 
            ", Reading time: ", 
            hit.entity.get("reading_time"), 
            ", Claps", hit.entity.get("claps")
        )

# Output:
# Matched IDs:  [442005795759615782, 442005795759615816, 442005795759613616]
# Distance to the query vector:  [0.36103832721710205, 0.3767401874065399, 0.4162980318069458]
# Matched articles: 
# Title:  The Hidden Side Effect of the Coronavirus , Reading time:  8 , Claps 83
# Title:  Why The Coronavirus Mortality Rate is Misleading , Reading time:  9 , Claps 2900
# Title:  Coronavirus shows what ethical Amazon could look like , Reading time:  4 , Claps 51
```

## What's next

[Supported data types](schema.md#Supported-data-type)
[Boolean express rules](boolean.md)
[JavaScript Object Notation (JSON)](json_data_type.md)