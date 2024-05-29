---
title: How To Create A dlt Source With A Custom Authentication Method (With Zoom Example)
categories: Python, dlt
subtitle: A Comparison Between dlt And The Airbyte Low-Code CDK to Build a Data Platform
thumbnail: /assets/img/posts/gear-wheels.jpg)
author: Willi Müller
---


# tldr;
In this case study, we conclude that the REST API Source toolkit is a promising component for a data platform because it allows rapid development of high-quality ELT data pipelines with hardly any code for people with medium programming experience.
Its declarative interface uses Python dictionaries instead of YAML or JSON like previous systems – this allows more advanced developers to inject custom functionality and even write their own authorization methods, e.g. different flavors of OAuth 2.0.
In this article, we demonstrate the implementation of a dlt source that imports meeting data from Zoom using a custom OAuth 2.0 authorization.
We compare our implementation with the implementation of the community-contributed Airbyte Zoom source created using the low-code CDK.

# Disclaimer
[We](https://untitleddata.company/) were dlthub's design partners for the development of the REST API Source toolkit. We were the first testers and also contributed to its code.
The impetus came from a client project where the company wanted to migrate two dozen connectors from a difficult-to-scale Airbyte open-source installation. Thus, they wanted a framework with two characteristics:
quick connector development with code that is easy to maintain
a system that would allow them to run the pipelines efficiently to address their issues of scale

The title image comes from [vecteezy.com](https://www.vecteezy.com/vector-art/111315-free-gear-wheels-vector)

# Background
Data ingestion is a core component of a data platform and it is important to understand how the characteristics of a system suit a particular organization.

[Dlthub recently released](https://dlthub.com/docs/blog/rest-api-source-client) a new REST API Source toolkit that promises high-level and Python-only development of ELT pipelines loading from REST APIs.

Intending to choose the data loading component for a data platform, we benchmark it and compare it with the most prominent prior work, the Airbyte low-code connector development kit (CDK).
We selected the Airbyte Low-code CDK as a standard of comparison because we used it in a data platform at a larger client company. There, it has been enabling backend developers to load their product data into the analytical database.

OAuth 2.0 is a common way to securely authorize.
However, many developers fear OAuth 2.0 because its implementations vary across API providers and thus it becomes complex and is re-implemented multiple times.
Because of these subtle differences, we need a flexible interface that allows customizations.

Thus, we want to benchmark how easily we can implement the specific OAuth 2.0 for Zoom to the benchmark.


# Reasons We Want It
The point of this article is to compare dlt's approach to REST API source development with the Airbyte Low-code CDK.

Our goal is not to develop an ETL solution to load Zoom's meeting and webinar data because there are already implementations, [such as Fivetran](https://fivetran.com/docs/connectors/applications/zoom) and [Airbyte](https://docs.airbyte.com/integrations/sources/zoom).

We use Zoom's API as a case study to evaluate how suitable the dlt REST API Source toolkit would be as a data platform component to build and maintain a multitude of source connectors.


# Challenges and Opportunities
The most prominent connector development toolkit is the Airbyte Low-Code CDK.
Therefore, we take it as a standard and compare the recently released dlt REST API Source toolkit.


## Strengths of Prior Work
Airbyte's low-code connector development kit looks promising because it lets us configure our custom source using a connector builder UI which produces a YAML configuration file.
We love that the team at Airbyte developed a solution to accelerate the development of new REST API connectors and we celebrate their achievements.
We have successfully used Airbyte with multiple clients and we have also seen that introducing the low-code CDK has enabled people with little data engineering experience to specify successfully running ETL connectors.
This enabled them to have greater ownership over their raw data imports and their full data value chain.
The low-code CDK can be an interesting choice for a data platform because it standardizes the repetitive connector code and can ease maintenance.
The team at Airbyte [wrote in their documentation](https://docs.airbyte.com/connector-development/config-based/low-code-cdk-overview) how they observed that "API source connectors constitute the overwhelming majority of connectors, they are also the most formulaic. API connector code almost always solves small variations of these problems":
1. Requesting various endpoints
2. Authentication
3. Pagination
4. Rate limiting
5. Schema description
6. Decoding of the response
7. Supporting incremental loads

Also, the Airbyte Low-Code CDK offers a graphical UI to configure and test the custom connector.
![Airbyte Connector Builder GUI](/assets/img/posts/airbyte-users-stream-2.png)


### Challenges using Prior Work
However, using the Airbyte low-code CDK we encountered not only the powerful advantages listed above but also the following challenges:

1. The YAML code is long and repetitive and thus tedious and error-prone to write by hand. Airbyte's connector builder GUI helps generate it – but along with the advantages of a GUI over code come also limitations, such as lack of automation and customization.
2. The inherent limitations of YAML make it cumbersome to customize or natively inject functionality with callables or reuse code to keep the implementation [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself)
3. Airbyte connectors run only on a full Airbyte platform installation. Thus, we need at least one VM. We've heard from multiple practitioners that they encountered difficulties while trying to scale open-source Airbyte on Kubernetes. In contrast, dlt is a library that [can be imported anywhere](https://dlthub.com/docs/walkthroughs/deploy-a-pipeline) – be it Github actions, a Lambda function, Airflow, or in Docker on Kubernetes.


### How the dlt REST API Toolkit solves these challenges
We use the declarative flavor of the [REST API Source Toolkit](https://dlthub.com/docs/dlt-ecosystem/verified-sources/rest_api), which allows us to configure a generic REST API Source with the specifics of the Zoom endpoints we are interested in.

1. We have a declarative and Pythonic pipeline
1. our IDE understands the code and helps with autocomplete, thanks to typed Python dictionaries
1. we can leverage the Python toolchain: linting, interactive debugging and stepping through the code, automated test suite, CI/CD, version control
2. we can include callables to insert functionality and we are not restricted to strings, lists, numbers, and dictionaries
2. we can reference and reuse code which makes our configuration DRY
2. We have much less code to maintain (229 lines of Python vs. Airbyte's 790 lines of YAML + 80 loc Python for Zoom OAuth2)
3. We can run it everywhere where dlt runs, that is everywhere you can import a Python library


## Implementing the dlt REST API Toolkit

### Step 1: Connecting Securely with OAuth 2 to Zoom
Dlt offers a generic OAuth 2.0 implementation of the two-legged flow which is commonly employed for server-to-server authorization without user consent.
To connect to the Zoom API we can customize it by implementing a subclass `OAuth2Zoom`.
The generic OAuth 2.0 implementation offers the template method `obtain_token()` which calls three different step methods that our subclass needs to implement with the details specific to the Zoom API.
```python
    def obtain_token(self) -> None:
        response = requests.post(**self.build_access_token_request())
        response.raise_for_status()
        response_json = response.json()
        self.access_token = self.parse_access_token(response_json)
        expires_in_seconds = self.parse_expiration_in_seconds(response_json)
        self.token_expiry = pendulum.now().add(seconds=expires_in_seconds)
```

Here follows our Zoom-specific implementation of building the access token request and parsing from the response the access token and its expiration time.
```python
from rest_api.auth import OAuth2ImplicitFlow

class OAuth2Zoom(OAuth2ImplicitFlow):
    def build_access_token_request(self) -> Dict[str, Any]:
        authentication: str = b64encode(f"{self.client_id}:{self.client_secret}".encode()).decode()
        return {
            "url": "https://zoom.us/oauth/token",
            "headers": {
                "Authorization": f"Basic {authentication}",
                "Content-Type": "application/x-www-form-urlencoded",
            },
            "data": self.access_token_request_data,
        }
```
For details, see the [OAuth 2.0 documentation for Zoom](https://developers.zoom.us/docs/api/rest/using-zoom-apis/#server-to-server-authentication).

With the OAuth token retrieval in place, we can now plug in this freshly written authentication class into the declarative REST API Source Toolkit.
Please note that by using `dlt.secrets` we use [dlt's built-in mechanism to handle secrets securely](https://dlthub.com/docs/general-usage/credentials/configuration). Never hardcode credentials in the source code!
It reads secrets from a credential store, environment variables, or a secrets file.
```python
import dlt

from rest_api import RESTAPIConfig, rest_api_source

config: RESTAPIConfig = {
    "client": {
        "base_url": "https://api.zoom.us/v2",
        "auth": OAuth2Zoom(
            access_token_request_data={
                "grant_type": "account_credentials",
                "account_id": dlt.secrets["sources.zoom.account_id"],
            },
            client_id=dlt.secrets["sources.zoom.client_id"],
            client_secret=dlt.secrets["sources.zoom.client_secret"],
        ),
    },
}
```

In comparison, this is how authentication is configured using the Airbyte low-code CDK.
Instead of our Python dictionary, we find the configuration in a very similar YAML.

```yaml
  requester:
    url_base: "https://api.zoom.us/v2"
    http_method: "GET"
    authenticator:
      class_name: source_zoom.components.ServerToServerOauthAuthenticator
      client_id: "{{ config['client_id'] }}"
      account_id: "{{ config['account_id'] }}"
      client_secret: "{{ config['client_secret'] }}"
      authorization_endpoint: "{{ config['authorization_endpoint'] }}"
      grant_type: "account_credentials"
```
The connector code also includes an [implementation of a Python class handling OAuth 2.0](https://github.com/airbytehq/airbyte/blob/751b7af4bb2c1e520055df08aff5da33e2e44052/airbyte-integrations/connectors/source-zoom/source_zoom/components.py) in a similar fashion requiring the pipeline user to pass in the `account_id`, `client_id`, and `client_secret` via Airbyte's secret backend.

While trying to reproduce the Zoom connector ourselves, we were also successful without the custom class and with the following YAML configuration.
To enter a custom grant type we needed to switch from the GUI to the YAML code.
```yaml
type: OAuthAuthenticator
refresh_request_body:
  account_id: '{{ config[''account_id''] }}'
token_refresh_endpoint: https://zoom.us/oauth/token
grant_type: account_credentials
client_id: '{{ config["client_id"] }}'
client_secret: '{{ config["client_secret"] }}'
```

### Step 2: Configuring Pagination

Having implemented the secure connection to the Zoom API, we can now declare the endpoints and the pagination method used by the Zoom API.

First, we start with the pagination:
```python
from rest_api.paginators import JSONResponseCursorPaginator

config: RESTAPIConfig = {
    # omitting the previously given configs for base_url and auth
    ...
    "client": {
        "paginator": JSONResponseCursorPaginator(
            cursor_path="response.next_page_token",
            cursor_param="next_page_token",
        ),
    },
```

In comparison, this is the pagination using the Airbyte Low-Code CDK:
```YAML
  zoom_paginator:
    type: DefaultPaginator
    pagination_strategy:
      type: "CursorPagination"
      cursor_value: "{{ response.next_page_token }}"
      stop_condition: "{{ response.next_page_token == '' }}"
      page_size: 30
    page_size_option:
      field_name: "page_size"
      inject_into: "request_parameter"
    page_token_option:
      type: RequestOption
      field_name: "next_page_token"
      inject_into: "request_parameter"
```

The `retriever` and `schema_loader` configured with the Airbyte Low-Code CDK are not necessary to configure for the dlt REST API Source.

### Step 3: Configuring Endpoints

#### The /users Endpoint
The first endpoint we'd like to load is the list of users.
With the dlt REST API Source, it looks as follows:
```python
config: RESTAPIConfig = {
    # omitting the previously given configs for client, base_url, auth, and config
    "resources": [ "users" ]
},
```
Our configuration contains only the string `"users"` because dlt here uses convention over configuration and assumes that this string corresponds to the endpoint path.
Also, it uses the paginator we declared as a default for the REST client and can automatically find the right strategy to extract the data from the response.
Further, dlt's core engine unpacks the JSON and infers the schema and data types.

In comparison, this is the stream loading the `/users` endpoint with the Airbyte Low-Code CDK:
```YAML
  users_stream:
    schema_loader:
      $ref: "#/definitions/schema_loader"
    retriever:
      paginator:
        $ref: "#/definitions/zoom_paginator"
      record_selector:
        extractor:
          type: DpathExtractor
          field_path: ["users"]
      $ref: "#/definitions/retriever"
    $parameters:
      name: "users"
      primary_key: "id"
      path: "/users"
```

#### Defining the Schema
Our dlt connector benefits from dlt's schema management engine.
This means, that dlt automatically:
- unpacks the JSON (normalization) into flat tables
- infers the data type of each field as well as primary keys
- optionally allows us to type the columns
- [automatically evolves the schema](https://dlthub.com/docs/general-usage/schema-evolution) according to changing data deliveries
- optionally allows us to reject data that does not conform to the ([data contracts](https://dlthub.com/docs/general-usage/schema-contracts))

The Airbyte Low-code CDK has similar capabilities in schema management.
Yet, we found two main differences.
First, the Airbyte Low-Code CDK can detect the JSON schema for each stream and then write it into a file which is part of the connector source code.
In contrast, dlt does not require us to keep the schema specification as part of the code but it keeps it in its private state metadata.
Second, as of now, Airbyte seems to have slightly fewer options to react to schema changes than dlt offers.

#### Dependent Resources: The /users/{user_id}/meetings Endpoint

To retrieve all meetings belonging to a user we need to configure a resource called `meetings`.
Meetings depend on the existing `users` resource because it resolves the `"user_id"` in the API path from the `users` resource.

```python
config: RESTAPIConfig = {
    # omitting the previously given configs for client, base_url, auth, and config
    "base_url": ...
    "auth": ...
    "client": ...
    "resources": [
        "users",  # parent resource
        {
            "name": "meetings",  # child resource
            "endpoint": {
                "path": "users/{user_id}/meetings",
                "params": {
                    "user_id": {
                        "type": "resolve",
                        "resource": "users",  # reference to the parent resource
                        "field": "id",
                    }
                },
            },
        },
    ]
}
```

In comparison, this is the implementation of the dependent stream using the Airbyte Low-Code CDK:
```yaml
  meetings_list_tmp_stream:
    schema_loader:
      $ref: "#/definitions/schema_loader"
    $parameters:
      name: "meetings_list_tmp"
      primary_key: "id"
    retriever:
      paginator:
        $ref: "#/definitions/zoom_paginator"
      record_selector:
        extractor:
          type: DpathExtractor
          field_path: ["meetings"]
      $ref: "#/definitions/retriever"
      requester:
        $ref: "#/definitions/requester"
        path: "/users/{{ stream_slice.parent_id }}/meetings"
      partition_router:
        type: SubstreamPartitionRouter
        parent_stream_configs:
          - stream: "#/definitions/users_stream"
            parent_key: "id"
            partition_field: "parent_id"
```
We noted that the specification of components, like the record extractor, record retriever, and paginator is repetitive because we could not find a way of defining shared configurations for all resources only once.
The dlt REST API Source, in contrast, allows us to configure commonly shared configurations only once in the [`"client"`](https://dlthub.com/docs/dlt-ecosystem/verified-sources/rest_api#client) config or the [`"resource_defaults"`](https://dlthub.com/docs/dlt-ecosystem/verified-sources/rest_api#resource_defaults-optional) respectively.
Additionally, we noted that the Airbyte Low-Code CDK requires a configuration of the schema and primary key for each stream which the connector builder UI can luckily extract from responses during development.
In contrast, it is part of dlt's core functionality to automatically infer the schema and manage schema evolution with optional data contracts.
Therefore, the schema configuration is not necessarily part of the source code.

Similarly, we continue defining all desired streams.
We noticed that what takes only a single Python string in a single line of code using dlt requires about 22 lines of YAML configuration using the Airbyte Low-Code CDK.
For dependent resources, we require about 12 lines of Python code using dlt and about 20-35 lines of YAML using the Airbyte Low-Code CDK.

Because we have multiple resources depending on the `user_id` we can extract the dictionary declaring the `user_id` to be resolved from the `id` field in the `users` resource:

```python
resolve_user_id = {
    "user_id": {
        "type": "resolve",
        "resource": "users",
        "field": "id",
    }
}
```

The `/users/{user_id}/meetings` resource configuration then shrinks down to:
```python
{
    "name": "meetings",
    "endpoint": {
        "path": "users/{user_id}/meetings",
        "params": resolve_user_id,
    },
},
```

See the full [Zoom dlt REST API source implementation here](https://github.com/untitled-data-company/dlt-rest-api-tutorial/blob/main/zoom.py).

In comparison, here is the full code produced with Airbytes low-code CDK via the connector builder GUI: [Airbyte Zoom Source](https://github.com/airbytehq/airbyte/blob/e669832b184d0e864a7b57343ee7d4ae3f285af1/airbyte-integrations/connectors/source-zoom/source_zoom/manifest.yaml).

#### Handling Errors
The Zoom API returns error codes in case a requested entity does not exist or a certain feature is not available.
Often, we do not want to crash our pipeline but gracefully ignore specific errors.
With `response_actions`, we [can define](https://dlthub.com/docs/dlt-ecosystem/verified-sources/rest_api#response-actions) what happens in case the HTTP response code is 400 and above.

For example, we want to ignore the error 404 in case the meeting has expired:
```python
{
  "name": "meeting_participants_report",
  "endpoint": {
      "path": "/report/meetings/{meeting_id}/participants",
      "params": resolve_meeting_id,
      "response_actions": [
          {"status_code": 404, "action": "ignore"},
      ],
  },
},
```
This was the complete endpoint configuration.

In comparison, this is only the error handling, not the entire stream implemented using the Airbyte Low-Code CDK:
```yaml
error_handler:
  type: CompositeErrorHandler
  error_handlers:
    - type: DefaultErrorHandler
      response_filters:
        - http_codes: [400]
          action: IGNORE
    - type: DefaultErrorHandler
```

Depending on the API design, we might not want to ignore all responses with the same error code but only specific error messages.
In the example below, we do not raise an exception if the response contains the substring "Registration has not been enabled for this meeting":
```python
"response_actions": [
    {
        "content": "Registration has not been enabled for this meeting",
        "action": "ignore",
    }
]
```

In comparison, this is how we ignore the same substring using the Airbyte Low-Code CDK:
```yaml
error_handler:
  type: CompositeErrorHandler
  error_handlers:
    - type: DefaultErrorHandler
      response_filters:
        - type: HttpResponseFilter
          action: IGNORE
          error_message_contains: Registration has not been enabled for this meeting
```

### Step 4: Writing the Pipeline
```python
import dlt
from zoom import source

pipeline = dlt.pipeline(
    pipeline_name="zoom_test", destination="duckdb", dataset_name="zoom", progress="log"
)
load_info = pipeline.run(source)
print(load_info)
```

## Conclusion
We conclude from this case study that the dlt REST API Toolkit is a great building block for a data platform because it supports the rapid development of high-quality custom source connectors.
This toolkit allows teams to ingest their domain-specific data in a way that is easy to maintain and easy to scale up and down.
We prefer it over the Airbyte Low-code CDK, especially when we need complex authentication or custom logic. Also, when want to take advantage of the Python toolchain, such as IDE support, interactive debugging, automated test suite, version control, etc.
We like that it inherits the advantages of dlt, such as automatic schema management and being embeddable in a lightweight manner instead of being a platform on its own because it plays together with existing schedulers and execution environment.
However, we understand that every data platform can have unique needs according to its users.
Thus, we might prefer the Airbyte Low-code CDK in case Airbyte is already being used and does not pose a scalability bottleneck.
Also, Airbyte is preferable if the connector developers prefer the GUI source builder over writing Python configuration dictionaries or Python code.

If you're considering implementing dlt or optimizing your existing data platform, [contact us](https://untitleddata.company/) to discuss your specific requirements. Together, we explore how we can help you leverage dlt or other technologies for more efficient and scalable data infrastructure.
