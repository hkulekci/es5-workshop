## Ingest Node

We can use logstash for manipulating out data while indexing our elastic 
cluster. With Elastic 5.x, we can also use *ingest node* to pre-process our
documents. 

Ingest node is enabled by default for all nodes. You can disable for some nodes,
and you can create your dedicated ingest node. Check the following 
configurations of `elasticsearch.yml`:

```
# dedicated ingest node configuration
node.data: false
node.master: false
node.ingest: true
```

### Pipeline

Pipelines are a series of processors that are executed to change your data or 
types. 

```
{
  "description" : "Your Pipeline Description",
  "processors" : [ ... ]
}
```

You can check pipelines that had already created on your systems with the below 
request.

```
# Get Ingest Pipelines
GET _ingest/pipeline
```

For our example, now we will create a pipeline to create us a date and latlon 
field from our messy data. 

```
# Create an Ingestion Pipeline
PUT _ingest/pipeline/gtd
{
  "description" : "Global Terrorist Database Pipeline",
  "processors" : [
    {
      "script": {
        "lang": "painless",
        "inline": "ctx.latlon = (ctx.latitude + ', ' + ctx.longitude)"
      }
    },
    {
      "script": {
        "lang": "painless",
        "inline": "String day = ctx.iday;\n if (ctx.iday == '0') { day = '01'; }\n String month = ctx.imonth;\n if (ctx.imonth == '0') { month = '01'; }\n ctx.date = ctx.iyear + '-' + month + '-' + day;"
      }
    },
    {
      "date" : {
        "field" : "date",
        "target_field" : "date",
        "formats" : ["yyyy-MM-dd"],
        "timezone" : "UTC"
      }
    }
  ]
}
```

In this example, you can easily see what we are doing. You can see on the 
request, our pipeline name is `gtd`. Then, in our pipeline, there is three 
processors.

First one will create us a `latlon` file from combininig of `latitude` and 
`longitude` fields. 

```
    {
      "script": {
        "lang": "painless",
        "inline": "ctx.latlon = (ctx.latitude + ', ' + ctx.longitude)"
      }
    }
```

Second processor will create a `date` field from bringing together some other 
fields. And there is some logic in this processor to be able to create a correct
`date` field.

```
    {
      "script": {
        "lang": "painless",
        "inline": "String day = ctx.iday;\n if (ctx.iday == '0') { day = '01'; }\n String month = ctx.imonth;\n if (ctx.imonth == '0') { month = '01'; }\n ctx.date = ctx.iyear + '-' + month + '-' + day;"
      }
    }
```

The last one will set a type for our new `date` field. We will ignore time part
of the date, in the example.

```
    {
      "date" : {
        "field" : "date",
        "target_field" : "date",
        "formats" : ["yyyy-MM-dd"],
        "timezone" : "UTC"
      }
    }
```

### Simulating Our Pipeline

We have created a pipeline for our Global Terrorism Dataset. Now, let's look 
our example to confirm it work correctly. In Elasticsearch, there is a simulate 
api interface. You should send your pipeline and documents like following 
example.

```
POST _ingest/pipeline/_simulate
{
    "pipeline": { ... },
    "docs": [
        {...},
        {...},
    ]
}
```

When we apply to our example: 

```
# Simulate the Pipeline
POST _ingest/pipeline/_simulate
{
  "pipeline" : {
    "description" : "Global Terrorist Database Pipeline",
    "processors" : [
      {
        "script": {
          "lang": "painless",
          "inline": "ctx.latlon = (ctx.latitude + ', ' + ctx.longitude)"
        }
      },
      {
        "script": {
          "lang": "painless",
          "inline": "String day = ctx.iday;\n if (ctx.iday == '0') { day = '01'; }\n String month = ctx.imonth;\n if (ctx.imonth == '0') { month = '01'; }\n ctx.date = ctx.iyear + '-' + month + '-' + day;"
        }
      },
      {
        "date" : {
          "field" : "date",
          "target_field" : "date",
          "formats" : ["yyyy-MM-dd"],
          "timezone" : "UTC"
        }
      }
    ]
  },
  "docs" : [
    {
      "_index": "index",
      "_type": "type",
      "_id": "id",
      "_source":{"iday": "02", "imonth":"0", "iyear":"2017", "latitude": "1", "longitude": "2"}}
  ]
}
```
