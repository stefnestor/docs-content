---
navigation_title: Logging action
mapped_pages:
  - https://www.elastic.co/guide/en/elasticsearch/reference/current/actions-logging.html
applies_to:
  stack: ga
products:
  - id: elasticsearch
---

# Logging action [actions-logging]

Use the `logging` action to log text to the standard Elasticsearch logs. See [Logging action attributes](#logging-action-attributes) for the supported attributes.

This action is primarily used during development and for debugging purposes.

## Configuring logging actions [configuring-logging-actions]

You configure logging actions in the `actions` array. Action-specific attributes are specified using the `logging` keyword.

The following snippet shows a simple logging action definition:

```js
"actions" : {
  "log" : { <1>
    "transform" : { ... }, <2>
    "logging" : {
      "text" : "executed at {{ctx.execution_time}}" <3>
    }
  }
}
```

1. The id of the action.
2. An optional [transform](transform.md) to transform the payload before executing the `logging` action.
3. The text to be logged.

## Logging action attributes [logging-action-attributes]

| Name | Required | Default | Description |
| --- | --- | --- | --- |
| `text` | yes | - | The text that should be logged. Can be static text or                                                          include Mustache [templates](how-watcher-works.md#templates). |
| `category` | no | xpack.watcher.actions.logging | The category under which the text will be logged. |
| `level` | no | info | The logging level. Valid values are: `error`, `warn`,                                                          `info`, `debug` and `trace`. |
