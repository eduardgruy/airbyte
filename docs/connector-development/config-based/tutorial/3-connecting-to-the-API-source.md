# Step 3: Connecting to the API

We're now ready to start implementing the connector.

Over the course of this tutorial, we'll be editing a few files that were generated by the code generator:

- `source-exchange-rates-tutorial/source_exchange_rates_tutorial/spec.yaml`: This is the [spec file](../../connector-specification-reference.md). It describes the inputs used to configure the connector.
- `source-exchange-rates-tutorial/source_exchange_rates_tutorial/exchange_rates_tutorial.yaml`: This is the connector definition. It describes how the data should be read from the API source.
- `source-exchange_rates-tutorial/integration_tests/configured_catalog.json`: This is the connector's [catalog](../../../understanding-airbyte/beginners-guide-to-catalog.md). It describes what data is available in a source
- `source-exchange-rates-tutorial/integration_tests/sample_state.json`: This is a sample state object to be used to test [incremental syncs](../../cdk-python/incremental-stream.md).

We'll also be creating the following files:

- `source-exchange-rates-tutorial/secrets/config.json`: This is the configuration file we'll be using to test the connector. Its schema should match the schema defined in the spec file.
- `source-exchange-rates-tutorial/secrets/invalid_config.json`: This is an invalid configuration file we'll be using to test the connector. Its schema should match the schema defined in the spec file.
- `source_exchange_rates_tutorial/schemas/rates.json`: This is the [schema definition](../../cdk-python/schemas.md) for the stream we'll implement.

## Updating the connector spec and config

Let's populate the specification (`spec.yaml`) and the configuration (`secrets/config.json`) so the connector can access the access key and base currency.

1. We'll add these properties to the connector spec in `source-exchange-rates-tutorial/source_exchange_rates_tutorial/spec.yaml`

```yaml
documentationUrl: https://docs.airbyte.io/integrations/sources/exchangeratesapi
connectionSpecification:
  $schema: http://json-schema.org/draft-07/schema#
  title: exchangeratesapi.io Source Spec
  type: object
  required:
    - access_key
    - base
  additionalProperties: true
  properties:
    access_key:
      type: string
      description: >-
        Your API Access Key. See <a
        href="https://exchangeratesapi.io/documentation/">here</a>. The key is
        case sensitive.
      airbyte_secret: true
    base:
      type: string
      description: >-
        ISO reference currency. See <a
        href="https://www.ecb.europa.eu/stats/policy_and_exchange_rates/euro_reference_exchange_rates/html/index.en.html">here</a>.
      examples:
        - EUR
        - USD
```

2. We also need to fill in the connection config in the `secrets/config.json`
   Because of the sensitive nature of the access key, we recommend storing this config in the `secrets` directory because it is ignored by git.

```bash
echo '{"access_key": "<your_access_key>", "base": "USD"}'  > secrets/config.json
```

## Updating the connector definition

Next, we'll update the connector definition (`source-exchange-rates-tutorial/source_exchange_rates_tutorial/exchange_rates_tutorial.yaml`). It was generated by the code generation script.
More details on the connector definition file can be found in the [overview](../low-code-cdk-overview.md) and [connection definition](../understanding-the-yaml-file/yaml-overview.md) sections.

Let's fill this out these TODOs with the information found in the [Exchange Rates API docs](https://apilayer.com/marketplace/exchangerates_data-api).

1. First, let's rename the stream from `customers` to `rates`, and update the primary key to `date`.

```yaml
streams:
  - type: DeclarativeStream
    $options:
      name: "rates"
    primary_key: "date"
```

and update the references in the `check` block

```yaml
check:
  type: CheckStream
  stream_names: [ "rates" ]
```

Adding the reference in the `check` tells the `check` operation to use that stream to test the connection.

2. Next we'll set the base url.
   According to the API documentation, the base url is `"https://api.apilayer.com"`.

```yaml
definitions:
  <...>
  retriever:
    type: SimpleRetriever
    $options:
      url_base: "https://api.apilayer.com"
```

3. We can fetch the latest data by submitting a request to the `/latest` API endpoint. This path is specific to the stream, so we'll set it within the `rates_stream` definition, at the `retriever` level.

```yaml
streams:
  - type: DeclarativeStream
    $options:
      name: "rates"
    primary_key: "date"
    schema_loader:
      $ref: "*ref(definitions.schema_loader)"
    retriever:
      $ref: "*ref(definitions.retriever)"
      requester:
        $ref: "*ref(definitions.requester)"
        path: "/exchangerates_data/latest"
```

4. Next, we'll set up the authentication.
   The Exchange Rates API requires an access key to be passed as header named "apikey".
   This can be done using an `ApiKeyAuthenticator`, which we'll configure to point to the config's `access_key` field.

```yaml
definitions:
  <...>
  requester:
    type: HttpRequester
    name: "{{ options['name'] }}"
    http_method: "GET"
    authenticator:
      type: ApiKeyAuthenticator
      header: "apikey"
      api_token: "{{ config['access_key'] }}"
```

5. According to the ExchangeRatesApi documentation, we can specify the base currency of interest in a request parameter. Let's assume the user will configure this via the connector configuration in parameter called `base`; we'll pass the value input by the user as a request parameter:

```yaml
definitions:
  <...>
  requester:
    <...>
    request_options_provider:
      request_parameters:
        base: "{{ config['base'] }}"
```

The full connector definition should now look like

```yaml
version: "0.1.0"

definitions:
  schema_loader:
    type: JsonSchema
    file_path: "./source_exchange_rates_tutorial/schemas/{{ options['name'] }}.json"
  selector:
    type: RecordSelector
    extractor:
      type: DpathExtractor
      field_pointer: [ ]
  requester:
    type: HttpRequester
    name: "{{ options['name'] }}"
    http_method: "GET"
    authenticator:
      type: ApiKeyAuthenticator
      header: "apikey"
      api_token: "{{ config['access_key'] }}"
    request_options_provider:
      request_parameters:
        base: "{{ config['base'] }}"
  retriever:
    type: SimpleRetriever
    $options:
      url_base: "https://api.apilayer.com"
    name: "{{ options['name'] }}"
    primary_key: "{{ options['primary_key'] }}"
    record_selector:
      $ref: "*ref(definitions.selector)"
    paginator:
      type: NoPagination

streams:
  - type: DeclarativeStream
    $options:
      name: "rates"
    primary_key: "date"
    schema_loader:
      $ref: "*ref(definitions.schema_loader)"
    retriever:
      $ref: "*ref(definitions.retriever)"
      requester:
        $ref: "*ref(definitions.requester)"
        path: "/exchangerates_data/latest"
check:
  type: CheckStream
  stream_names: [ "rates" ]
```

We can now run the `check` operation, which verifies the connector can connect to the API source.

```bash
python main.py check --config secrets/config.json
```

which should now succeed with logs similar to:

```
{"type": "LOG", "log": {"level": "INFO", "message": "Check succeeded"}}
{"type": "CONNECTION_STATUS", "connectionStatus": {"status": "SUCCEEDED"}}
```

## Next steps

Next, we'll [extract the records from the response](4-reading-data.md)

## More readings

- [Config-based connectors overview](../low-code-cdk-overview.md)
- [Authentication](../understanding-the-yaml-file/authentication.md)
- [Request options providers](../understanding-the-yaml-file/request-options.md)
- [Schema definition](../../cdk-python/schemas.md)
- [Connector specification reference](../../connector-specification-reference.md)
- [Beginner's guide to catalog](../../../understanding-airbyte/beginners-guide-to-catalog.md)