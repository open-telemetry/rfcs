# A Dynamic Configuration Service for the SDK

This proposal is for an experiment to configure metric collection periods. Per-metric and tracing configuration is also intended to be added, with details left for a later iteration.

It is related to [this pull request](https://github.com/open-telemetry/opentelemetry-proto/pull/155)

## Motivation

During normal use, users may wish to collect metrics every 10 minutes. Later, while investigating a production issue, the same user could easily increase information available for debugging by reconfiguring some of their processes to collect metrics every 30 seconds. Because this change is centralized and does not require redeploying with new configurations, there is lower friction and risk in updating the configurations.

## Explanation

This OTEP is a request for an experimental feature [open-telemetry/opentelemetry-specification#62](https://github.com/open-telemetry/opentelemetry-specification/pull/632). It is intended to seek approval to develop a proof of concept. This means no development will be done inside either the Open Telemetry SDK or the collector. Since this will be implemented in [opentelemetry-go-contrib](https://github.com/open-telemetry/opentelemetry-go-contrib) and [opentelemetry-collector-contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib), all of this functionality will be optional.

The user, when instrumenting their application, can configure the SDK with the endpoint of their remote configuration service, the associated Resource and a default config we revert to if we fail to read from the configuration service.

The user must then set up the config service. This MUST be done through the collector, which can be set up to expose an arbitrary configuration service implementation. Depending on implementation, this allows the collector to either act as a stand-alone configuration service, or as a bridge to remote configurations of the user's monitoring and tracing backend by 'translating' the monitoring backend's protocol to comply with the Open Telemetry configuration protocol.

## Internal details

In the future, it is intended to add per-metric configuration. For example, this would allow the user to collect 5xx server error counts ever minute, and CPU usage statistics every 10 minutes. The remote configuration protocol was designed with this in mind, meaning that it includes more details than simply the metric collection period.

Our remote configuration protocol will support this call:

```
service DynamicConfig {
  rpc GetConfig (ConfigRequest) returns (ConfigResponse);
}
```

A request to the config service will look like this:

```
message ConfigRequest{

  // Required. The resource for which configuration should be returned.
  opentelemetry.proto.resource.v1.Resource resource = 1;

  // Optional. The value of ConfigResponse.fingerprint for the last configuration
  // a resource received that was successfully applied.
  bytes last_known_fingerprint = 2;
}
```

While the response will look like this:

```
message ConfigResponse {

  // Optional. The fingerprint associated with this ConfigResponse. Each change
  // in configs yields a different fingerprint.
  bytes fingerprint = 1;

  // Dynamic configs specific to metrics
  message MetricConfig {


    // A Schedule is used to apply a particular scheduling configuration to
    // a metric. If a metric name matches a schedule's patterns, then the metric
    // adopts the configuration specified by the schedule.

    message Schedule {

      // A light-weight pattern that can match 1 or more
      // metrics, for which this schedule will apply. The string is used to
      // match against metric names. It should not exceed 100k characters.
      message Pattern {
        oneof match {
          string equals = 1;       // matches the metric name exactly
          string starts_with = 2;  // prefix-matches the metric name
        }
      }

      // Metrics with names that match at least one rule in the inclusion_patterns are
      // targeted by this schedule. Metrics that match at least one rule from the
      // exclusion_patterns are not targeted for this schedule, even if they match an
      // inclusion pattern.

      // For this iteration, since we only want one Schedule that applies to all metrics,
      // we will not check the inclusion_patterns and exclusion_patterns.
      repeated Pattern inclusion_patterns = 1;
      repeated Pattern exclusion_patterns = 2;

      // CollectionPeriod describes the sampling period for each metric. All
      // larger units are divisible by all smaller ones.
      enum CollectionPeriod {
        NONE = 0;  // For non-periodic data (client sends points whenever)
        SEC_1 = 1;
        SEC_5 = 5;
        SEC_10 = 10;
        SEC_30 = 30;
        MIN_1 = 60;
        MIN_5 = 300;
        MIN_10 = 600;
        MIN_30 = 1800;
        HR_1 = 3600;
        HR_2 = 7200;
        HR_4 = 14400;
        HR_12 = 43200;
        DAY_1 = 86400;
        DAY_7 = 604800;
      }
      CollectionPeriod period = 3;

      // Optional. Additional opaque metadata associated with the schedule.
      // Interpreting metadata is implementation specific.
      bytes metadata = 4;
    }

    // For this iteration, since we only want one Schedule that applies to all metrics,
    // we will have a restriction that schedules must have a length of 1, and we will
    // not check the patterns when we apply the collection period.
    repeated Schedule schedules = 1;

  }
  MetricConfig metric_config = 2;

  // Dynamic configs specific to trace, like the sampling rate of a resource.
  message TraceConfig {
    // TODO: unimplemented
    // This proposal focuses on the metrics portion, but tracing is a natural
    // candidate for something that might be useful to configure in the future,
    // so this field will be left here for that eventuality.
  }
  TraceConfig trace_config = 3;

  // Optional. The client is suggested to wait this long (in seconds) before
  // pinging the configuration service again.
  int32 suggested_wait_time_sec = 4;
}
```

The SDK will periodically read a config from the service using GetConfig. If it fails to do so, it will just use either the default config or the most recent successfully read config. If it reads a new config, it will apply it.

Export frequency from the SDK depends on Schedules. There can only be one Schedule for now, which defines the schedule for all metrics. The schedule has a CollectionPeriod, which defines how often metrics are exported.

In the future, we will add per-metric configuration. Each Schedule also has inclusion_patterns and exclusion_patterns. Any metrics that match any of the inclusion_patterns and do not match any of the exclusion_patterns will be exported every CollectionPeriod (ie. every minute). A component will be added that can export metrics that match a pattern from one Schedule at that Schedule's collection period, while exporting other metrics that match the patterns of another Schedule at that other Schedule's collection period.

The collector will support a new interface for a DynamicConfig service that can be used by an SDK, allowing a custom implementation of the configuration service protocol described above, to act as an optional bridge between an SDK and an arbitrary configuration service. This interface can be implemented as a shim to support accessing remote configurations from arbitrary backends. The collector is configured to expose an endpoint for requests to the DynamicConfig service, and returns results on that endpoint.

## Trade-offs and mitigations

This feature will be implemented purely as an experiment, to demonstrate its viability and usefulness. More investigation can be done after a rough prototype is demonstrated.

As mentioned [here](https://github.com/open-telemetry/opentelemetry-proto/pull/155#issuecomment-640582048), the configuration service can be a potential attack vector for an application instrumented with Open Telemetry, depending on what we allow in the protocol. This can be mitigated since this proposal only allows the SDK to read from a configuration service managed by the collector. Additionally, 
We can highlight in the remote configuration protocol that for future changes, caution is needed in terms of the sorts of configurations we allow. 

## Prior art and alternatives

One way to configure metric export schedules is to, instead of pushing the metrics, use a pull mechanism to have metrics pulled whenever wanted. This has been implemented for [Prometheus](https://github.com/open-telemetry/opentelemetry-go/pull/751).

The alternative is to stick with the status quo, where the SDK has a [fixed collection period](https://github.com/open-telemetry/opentelemetry-go/blob/34bd99896311a81cf843475779cae2e1c05e6257/sdk/metric/controller/push/push.go#L72-L76) and a fixed sampling rate.

## Open questions

- As mentioned [here](https://github.com/open-telemetry/opentelemetry-proto/pull/155#issuecomment-640582048). what happens if a malicious/accidental config change overwhelms the application/monitoring system? Is it the responsibility of the user to be cautious while making config changes? Should we automatically decrease telemetry exporting if we can detect performance problems?

## Future possibilities

If this OTEP is implemented, there is the option to remotely and dynamically configure other things. As mentioned by commenters in the associated pull request, possibilities include labels and aggregations.

It is intended to add per-metric configuration and well as tracing configuration in the future.