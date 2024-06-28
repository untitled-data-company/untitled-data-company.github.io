---
title: dlt REST API recipes - Query an endpoint with a list of values
categories: Python, dlt
subtitle: Leveraging Python, keep the code simple, live happy
thumbnail: /assets/img/posts/food-preparation.jpg
author: Francesco Mucio
---

## Reding Requirements

This post will be easier to read if you are familiar with:
- a basic knowledge of [Python](https://www.python.org/) lists and list comprehension.
- [dlt](https://dlthub.com) - a Python library to move data between (many) sources to (many) destination.
- the dlt [REST API source](https://dlthub.com) - a dlt source to ingest data from REST APIs in a declarative way. [Here](https://www.youtube.com/watch?v=rdNj2S3lli0&ab_channel=UntitledDataCompany) a quick recap.
- some interest in getting data from a REST API.

 
# Python > YAML 
When we started discussing a declarative way to ingest REST API data with dlt, one thing was clear in my mind: we would stick to Python. Sure, someone will soon come up with a YAML version of it, but that's fine. The YAML version will be a wrapper and, like every wrapper, it will limit the available features (making other things simpler). Wrapping is perfect to give someone a gift like a chocolate box or for making a certain feature easier to use.

If you dealt enough with YAML (or any DSL), you will try to write/generate it using a different tool or language. It is a constant struggle. The big problem is that you cannot go lower than the features exposed by YAML, you cannot see what is underneath it.

Better stick to Python. You can still become a [YAML engineer](https://www.youtube.com/watch?v=3L_HoTT3A6U) later.

# The problem - Passing a list of values to a query parameter
Recently someone in the [dlt Slack](https://join.slack.com/t/dlthub-community/shared_invite/zt-1n5193dbq-rCBmJ6p~ckpSFK4hCF2dYA) asked for a way to get data from an API endpoint passing multiple values as a query parameter. 

Imagine you are a customer centric business, everyday you get a list of users who celebrate their birthday today, and you need to send them flowers; to do this you can use the `users?id={id}` endpoint to get their address. Because you are a successful business, you have many customers and you need to make many API calls.

Now if you are familiar with the dlt REST API source, you know that we can pass a value to [a query parameter like this](https://www.youtube.com/watch?v=rdNj2S3lli0&t=718):

```python
    config_object = {
        ...
        "resources": [
            {
                "name": "users",
                "endpoint":{
                    "params": {
                        "id": 2
                    },
                }
            }
        ]
        ...
    }
```
The value `2` will be passed as a query parameter to the endpoint and it will build an url like `users?id=2`. This works well for a single value. But what if we need to pass multiple values (`ids = [1, 2, 3]`) and the endpoint accepts only atomic values (`1` or `2` or `3`)?

# Looking for a solution
Our first idea would be to submit a feature request for dlt: when a list is passed, the REST API source will use all the values of the list. This sounds good, but let's look into that.

What if, for the same endpoint, we have two query parameters with a list of values (`id1 = [1, 2, 3]` and `id2=["a", "b", "c"]`)? 

Should we use them in parallel (`(1, "a")`, `(2, "b")`, `(3, "c")`) or should we do a Cartesian product (`(1, "a")`, `(1, "b")`, `(1, "c")`, `(2, "a")`, and so on)? Taking one decision instead of another could cause problems downhill. 

Another thing to consider is how the REST API source will call the endpoint: since the endpoint can accept only a single value, there is no alternative, there will be 3 different call:

```
HTTP GET .../users?id=1
HTTP GET .../users?id=2
HTTP GET .../users?id=3
```

So we can summarize the intial solution (passing a list) as:

\+ easy to write for  developer

\+/- not changing much in terms of performance

\- limiting some of the possible use cases  

Still using a list is nice...

# Python to the rescue
Because the config object for the REST API is a Python object, we can also assemble it with smaller building blocks. For example: first, let's get that list of IDs and create a list of resources:

```python
    from rest_api import  EndpointResource
    
    ids = [1, 2, 3]

    users_resources: List[EndpointResource] = [
        {
            "name": f"users_{id}",
            "endpoint": {
                "path": f"users",
                "params": {
                    "id": id
                },
            },
            "table_name": "users",
        }
        for id in ids
    ]
```

In this way we will have a resource for each value in the IDs list; each resource will have it's own name, but they will all write in the same table.

And the config object will look like this:

```python
    config_object = {
        ...
        "resources": users_resources
        ...
    }
```
Keeping it quite compact.

This approach allows for the generation of multiple endpoint resources programmatically, accommodating various use cases, such as passing lists of query parameters, and even defining how values from different lists can be combined (I leave this to use as homework).

# Wrapping up
We started with an interesting problem, we explored a possible solution (accepting a list as param value), just to figure out that the solution was already there in fron of us.

I cannot hide my preference to write Python code because it allows more sophisticated configurations than what is possible with static DSLs like YAML.

This flexibility not only simplifies the development process but also ensures that our configuration remains adaptable to changing requirements. Embracing Python for this task enhances both the usability and scalability of the dlt REST API source, making data integration tasks more efficient and effective.

# Bonus Use Case: Generated resources with child resources
What if for each endpoint result we need also to call a secondary endpoint to get some additional information.

Let's assume we have an endpoint returning all the articles for a certain category, `articles?category={category_id}`, and a second to the articles details, `articles/{article_id}/details`.

The REST API source config object provides [a way to define relationships](https://dlthub.com/docs/dlt-ecosystem/verified-sources/rest_api#define-resource-relationships) between resource.

<details>
  <summary><i>You can see how opening this hidden section ;)
  
  </i></summary>
  ## REST API related resources
    For a single category we could do something like this:

```python
    config_object = {
        ...
        "resources": [
            {
                "name": "articles_by_category",
                "endpoint":{
                    "path": "articles",
                    "params": {
                        "category": 481
                    },
                }
            },
            {
                "name": "articles",
                "endpoint":{
                    "path": "articles/{article_id}/details",
                    "params": {
                        "article_id": {
                            "type": "resolve",
                            "resource": "articles_by_category",
                            "field": "id",
                        },
                    },
                }
            },
        ]
        ...
    }
```
</details>

But if you have a number of categories for which you need articles this can become a bit too much. Let's do it with Python:

```python  
    category_ids = [481, 122, 839, ...]

    articles_by_category: List[EndpointResource] = [
        {
            "name": f"articles_by_category_{category_id}",
            "endpoint": {
                "path": f"articles",
                "params": {
                    "category": category_id
                },
            },
            "table_name": "articles_by_category",
        }
        for category_id in category_ids
    ]
    
    articles: List[EndpointResource] = [
        {
            "name": f"articles_of_category_{category_id}",
            "endpoint": {
                "path": f"articles/{article_id}/details",
                "params": {
                    "article_id": {
                        "type": "resolve",
                        "resource": f"articles_by_category_{category_id}",
                        "field": "id",
                    },
                },
            },
            "table_name": "articles",
        }
        for category_id in category_ids
    ]
```
Now we need only to add both the generated resources to the config object. Well, [in Python joining two lists](https://stackoverflow.com/questions/1720421/how-do-i-concatenate-two-lists-in-python) if just `+` away:

```python
    config_object = {
        ...
        "resources": articles_by_category + articles
        ...
    }
```
