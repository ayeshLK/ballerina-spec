# 1439: HTTP Client Retry Reset Support

- Authors - @ayeshLK
- Reviewed by - 
- Created date - 2026-03-09
- Updated date - 2026-03-09
- Issue - [https://github.com/ballerina-platform/ballerina-spec/issues/1439](https://github.com/ballerina-platform/ballerina-spec/issues/1439)
- State - Submitted

## Summary
This proposal introduces a mechanism for the Ballerina HTTP client to reset its retry attempts upon exhaustion, facilitating high-resiliency scenarios such as continuous retrying for long-running tasks. This is achieved by extending the `http:RetryConfig` record with a `resetOnExhaust` field and implementing an internal `ResettableRetryClient` to manage the lifecycle of retry cycles.

## Motivation
In distributed systems and cloud-native environments, services are susceptible to transient failures and prolonged outages caused by network instability, resource pressure, or scheduled maintenance. Ballerina's current HTTP client retry mechanism is governed by a fixed `count`, which, once reached, results in a terminal failure. 

While a finite retry limit is appropriate for most interactive requests, it is insufficient for background processes, data synchronization tasks, or integration workflows that must eventually succeed despite extended downtime. Currently, developers must implement manual, boilerplate-heavy loops to achieve infinite or cycle-based retrying, which bypasses the standard, optimized retry infrastructure and reduces code readability.

Providing a native way to reset the retry counter empowers developers to build more resilient applications that can adapt to varying levels of service availability without sacrificing the simplicity and performance of the built-in HTTP client.

## Goals
- Enable the HTTP client to restart its retry cycle automatically once the maximum attempt count is reached.
- Introduce a standardized configuration option within `http:RetryConfig` to control this behavior.
- Maintain full backward compatibility with existing retry configurations.
- Ensure the internal implementation remains decoupled from core request-response processing.

## Non-Goals
- Modifying the default retry behavior (the default remains finite).

## Design

### Configuration Changes
The `http:RetryConfig` record will be updated to include the `resetOnExhaust` boolean field.

```ballerina
public type RetryConfig record {|
    int count = 0;
    decimal interval = 0;
    float backOffFactor = 0;
    decimal maxWaitInterval = 0;
    StatusCodes[] statusCodes = [];
    boolean resetOnExhaust = false;
|};
```

- **`resetOnExhaust`**: When set to `true`, the internal retry counter is reset to zero after the `count` attempts are exhausted. This triggers a new cycle of retries, effectively allowing the client to continue its attempts indefinitely or until the request succeeds.

### Internal Implementation: `ResettableRetryClient`

The feature will be implemented by introducing an internal `ResettableRetryClient`, which functions as a decorator over the standard `RetryClient`. This approach maintains a clean separation of concerns, allowing the existing retry logic to remain unchanged while extending it with cycle-reset capabilities.

### Example Usage
```ballerina
import ballerina/http;

// Configured for high-resiliency background sync
http:Client syncClient = check new ("http://api.sync-service.com",
    retryConfig = {
        count: 10,
        interval: 2,
        backOffFactor: 2.0,
        // Enables resetting the retry attempts once the configured retry limit is exhausted
        resetOnExhaust: true
    }
);
```

## Alternatives
- **Infinite Count Indicator**: Using a value like `-1` for `count` to signify infinity. This was rejected because cycle-based resetting provides better hooks for future observability (e.g., logging "End of Cycle 1") and maintains the semantic meaning of "retry count" as a discrete batch.
- **Manual Application Logic**: Encouraging users to wrap calls in `while` loops. This was rejected as it leads to fragmented error handling patterns and prevents the use of built-in features like backoff and status-code-based filtering.

## Testing
- **Cycle Reset Validation**: Unit tests to verify the internal counter correctly transitions from `max_count` back to `1` when `resetOnExhaust` is enabled.
- **Resiliency Scenarios**: Integration tests using a mock server that remains down for a duration exceeding the initial `count * interval` to ensure the client eventually succeeds after several reset cycles.
- **Regression Testing**: Verify that requests still fail terminally when `resetOnExhaust` is `false` (default behavior).

## Risks and Assumptions
- **Resource Management**: Continuous retrying can lead to excessive log generation or resource consumption if the `interval` is not configured appropriately.
- **Assumption**: It is assumed that users configuring `resetOnExhaust` understand the implications of potentially infinite request loops in their specific system architecture.
