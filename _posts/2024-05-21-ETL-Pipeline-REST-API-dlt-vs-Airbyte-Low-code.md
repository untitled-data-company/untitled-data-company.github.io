# How To Create A Dlt Source With A Custom Authentication Method
# Alt 2: ELT Pipeline from OAuth REST API
# Alt 3: Create dlt Source for REST API with OAuth 2.0 using REST API Source Toolkit
## A Comparison Between Dlt And The Airbyte Low-Code CDK to Build a Data Platform

## tldr;
The REST API Source toolkit is a promising component for a data platform because it allows rapid development of high-quality ELT data pipelines with hardly any code for people with medium programming experience.
Its declarative interface uses Python dictionaries instead of YAML or JSON like previous systems – this allows more advanced developers to inject custom functionality and even write own authorization methods, e.g. different flavors of OAuth 2.0.
In this article, we demonstrate the implementation of a dlt source that imports webinar data from Zoom using a custom OAuth 2.0 authorization.
We compare our implementation with the implementation of the Airbyte Zoom source which was achieved using the low-code CDK.

## Background
Data ingestion is a core component of a data platform and it is important understand how the characteristics of a system suit a particular organization.

[Dlthub recently released](https://dlthub.com/docs/blog/rest-api-source-client) a new REST API Source toolkit which promises high-level and Python-only development of ELT pipelines loading from REST APIs.

With the goal of choosing the data loading component for a data platform in mind, we benchmark it and compare with the most prominent prior work, the Airbyte low-code connector development kit (CDK).
We selected the Airbyte Low-code CDK as a standard of comparison because we used it in a data platform at a larger client company to  enable backend developers to load their product data into the analytical database.

OAuth 2.0 is a common way to securely authorize with a server.
However, many developers fear OAuth 2.0 because its implementations vary across API providers and thus it becomes complex and is re-implemented multiple times.
Because of these subtle differences we need a flexible interface which allows customizations.

Thus, we added an implementation of OAuth 2.0 for Zoom to the benchmark.


## Reasons we want it
We want to track the performance of our webinars or meetings and extract data, such as count of participants, contact data of webinar participants, duration of attendance, polls, reasons participants left etc.

There are already ELT solutions to load Zoom's Webinar data, [such as Fivetran](https://fivetran.com/docs/connectors/applications/zoom) and [Airbyte](https://docs.airbyte.com/integrations/sources/zoom) and thus the point of this article is not to reinvent the wheel but to compare dlt's approach to REST API source development with the Airbyte Low-code CDK.

We use Zoom's API as a case study to evaluate how suitable this would be as a data platform component.


## Challenges
Airbyte's low-code connector development kit looks promising because it lets us configure our own source using a connector builder UI which produces a YAML configuration file.
We love that the team at Airbyte developed a solution to accelerate the development of new REST API connectors and we celebrate their achievements.
We have successfully used Airbyte at multiple clients and we have also seen that introducing the low-code CDK has enabled people with little data engineering experience to specify successfully running ETL connectors.
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

However, using the low-code CDK we encountered not only the powerful advantages listed above but also the following challenges:
1. The YAML code is long and repetitive and thus tedious and error-prone to write by hand. Airbyte's connector builder GUI helps generating it but it has not only the advantages but also limitations of a GUI over code, such as lack of automation and customization.
2. The inherent limitations of YAML make it cumbersome to natively inject funtionality with callables or reuse code to keep the implementation [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself)
3. Airbyte connectors run only on a full Airbyte platform installtion. Thus, we need at least one VM. Scaling open-source Airbyte on Kubernetes has shown to be difficult. In contrast, dlt is a library that [can be imported anywhere](https://dlthub.com/docs/walkthroughs/deploy-a-pipeline) – be it github actions, a Lambda function, Airflow, or in Docker on Kubernetes.
4. Airbyte's low-code Zoom implementation does not support incremental syncing yet.


## Our Solution
We use the declarative flavor of the [REST API Source Toolkit](https://dlthub.com/docs/dlt-ecosystem/verified-sources/rest_api), which allows us to configure a generic REST API Source with the specifics of the Zoom endpoints we are interested in.


## Things unlocked now
1. We have a declarative and pythonic pipeline. This has a series of advantages:
  - no special language syntax to learn
  - our IDE understands the code and helps with autocomplete, thanks to typed dictionaries
  - we can leverage the Python tool chain: linting, interactive debugging and stepping through the code, automated test suite, CI/CD, version control
  - we can include callables to insert functionality and we are not restricted to strings, lists, numbers, and dictionaries
  - we can reference and reuse code which makes our configuration DRY
2. We have much less code to maintain (x lines of Python vs. Airbyte's Y lines of YAML + 80 loc Python for Zoom OAuth2)
3. We can run it everywhere where dlt runs, that is everywhere you can import a Python library
4. We can load data incrementally


## The Solution

### Step 1: Connecting Securely with OAuth 2 to Zoom
Dlt offers a generic OAuth 2.0 implementation of the implicit flow without which is commonly employed for server-to-server authorization without user consent.
To connect to the Zoom API we can customize it by implementing a subclass `OAuth2Zoom`.
The generic OAuth 2.0 implementation offers the template method `obtain_token()` which calls three step methods that our subclass needs to implement with the details specific to the Zoom API.
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
            "url": self.access_token_url,
            "headers": {
                "Authorization": f"Basic {authentication}",
                "Content-Type": "application/x-www-form-urlencoded",
            },
            "data": self.access_token_request_data,
        }

    def parse_access_token(self, response_json: Any) -> TSecretStrValue:
        return str(response_json.get("access_token"))

    def parse_expiration_in_seconds(self, response_json: Any) -> int:
        return int(response_json.get("expires_in", self.default_token_expiration))
```
For details, see the [OAuth 2.0 documentation for Zoom](https://developers.zoom.us/docs/api/rest/using-zoom-apis/#server-to-server-authentication).

With the OAuth token retrieval in place, we can now plug in this freshly written authentication class into the declarative REST API Source Toolkit:
```python
import dlt

from rest_api import RESTAPIConfig, rest_api_source

config: RESTAPIConfig = {
    "client": {
        "base_url": "https://api.zoom.us/v2",
        "auth": OAuth2Zoom(
            access_token_url="https://zoom.us/oauth/token",
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
Instead of our Python dictionary we find the configuration in very similar YAML.

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

#### The /Users Enpoint
The first endpoint we'd like to load is the list of users.
With the dlt REST API Source it looks as follows:
```python
config: RESTAPIConfig = {
    # omitting the previously given configs for client, base_url, auth, and config
    "resources": [ "users" ]
},
```
Our configuration contains only of the string `"users"` because dlt here uses convention over configuration and assumes that this string corresponds to the endpoint path.
Also, it uses the paginator we declared as a default for the REST client and can automatically find the right strategy to extract the data from the response.
Further, dlt's core engine unpacks the JSON and infers the schema and data types.

In comparison, this is the stream loading the users endpoint with the Airbyte Low-Code CDK:
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

#### The /users/{user_id}/meetings Stream

To retrieve all meetings belonging to a user we need to configure a resource called `meetings`.
Meetings depends on the existing `users` resource because it resolves the `"user_id"` in the API path from the `users` resource.

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
                        "resource": "users",  # reference to parent resource
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
We noted that the specification of components, like the record extractor, record retriever, and paginator is repetitive because there seems to be no way of defining shared configurations for all resources only once.
The dlt REST API Source, in contrast, allows us to configure commonly shared configurations only once in the [`"client"`](https://dlthub.com/docs/dlt-ecosystem/verified-sources/rest_api#client) config or in the [`"resource_defaults"`](https://dlthub.com/docs/dlt-ecosystem/verified-sources/rest_api#resource_defaults-optional) respectively.
Additionally, we noted that the Airbyte Low-Code CDK requires a configuration about the schema and primary key for each stream.
In contrast, it is part of dlt's core functionality to  automatically infer the schema and manage schema evolution with optional data contracts.

In a similar fashion, we continue defining all desired streams.
We noticed that what takes only a single Python string in a single line of code using dlt requires about 22 lines of YAML configuration using the Airbyte Low-Code CDK.
For dependent resources we require about 12 lines of Python code using dlt and about 20-35 lines of YAML using the Airbyte Low-Code CDK.

See the full dlt REST API source implementation here.

In comparison, here is the full code produced with Airbytes low-code CDK via the connector builder GUI: [Airbyte Zoom Source](https://github.com/airbytehq/airbyte/blob/e669832b184d0e864a7b57343ee7d4ae3f285af1/airbyte-integrations/connectors/source-zoom/source_zoom/manifest.yaml).


### Step 4: Writing the Pipeline
```python
import dlt
from zoom import source

pipeline = dlt.pipeline(pipeline_name="zoom_test", destination="duckdb", progress="log")
load_info = pipeline.run(source.with_resources("users", "meetings"))
print(load_info)
```

## Conclusion
TODO: repeat the things unlocked and tie it back to data platform

We conclude that the dlt REST API Source toolkit can be an excellent choice for the data loading component in a data platform.
We like best that...
However, we might prefer the Airbyte Lowc-code CDK in case ...
