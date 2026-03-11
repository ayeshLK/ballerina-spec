# 1441: WebSub/WebSubHub Ping Semantic

- Author: @ayeshLK
- Reviewers: 
- Created: 2026-03-11
- Updated: 2026-03-11
- Issue: 
- Status: Submitted

## Summary

This proposal introduces a "ping" mechanism between the WebSub Hub and its Subscribers to verify the health of active subscriptions. It adds a new `ping()` remote function to the `websubhub:HubClient`, allowing Hubs to perform periodic health checks on subscribers. This helps in identifying and cleaning up "stale" subscriptions where the subscriber is no longer reachable or has been terminated without formal unsubscription.

## Motivation

In large-scale WebSub deployments, managing subscription lifecycles effectively is critical. Currently, if a subscriber becomes unreachable or its service is terminated without sending an unsubscription request, the Hub continues to treat the subscription as active. This leads to:
1.  **Stale Subscriptions**: Hubs attempt to deliver notifications to non-existent or unreachable endpoints, wasting resources.
2.  **Operational Overhead**: Manual intervention is often required to identify and remove these stale entries.
3.  **Inconsistency**: Subscribers that recover might remain inactive until they manually re-subscribe if the Hub has internally flagged them after repeated delivery failures, but lacks a formal way to verify their status.

By introducing a lightweight ping semantic, Hubs can proactively verify subscriber health using a standard HTTP-based mechanism.

## Goals

- Introduce a `ping()` remote function in the `websubhub:HubClient`.
- Define a standard HTTP GET protocol for pinging subscribers using the `hub.mode=ping` parameter.
- Define how Hubs should interpret subscriber responses (2xx, 410, etc.) to manage subscription status.

## Non-Goals

- Implementing an automatic background cleanup task in the WebSubHub module (the proposal provides the mechanism, but the cleanup logic remains with the Hub implementation).
- Modifying the core WebSub W3C recommendation (this is an extension for enhanced operability).

## Design

The proposal involves two main components: the client API change and the protocol definition.

### Client API Change

The `websubhub:HubClient` will be updated to include a new `ping` remote function:

```ballerina
public isolated client class HubClient {
    // ... existing functions

    # Pings the subscriber to verify health.
    #
    # + return - `()` if the subscriber is healthy, `websubhub:SubscriptionDeletedError` if subscriber returned HTTP 410 GONE, or `websubhub:Error` if unreachable or stale
    remote function ping() returns websubhub:SubscriptionDeletedError|websubhub:Error?;
}
```

### Protocol Definition

When `ping()` is invoked, the `HubClient` will send an HTTP GET request to the subscriber's callback URL.

**Request Details:**
- **Method**: `GET`
- **Query Parameters**:
    - `hub.mode`: `ping`
    - `hub.topic`: (Optional) The topic associated with the subscription.

**Response Interpretation:**

| HTTP Status Code | Interpretation | Action |
| :--- | :--- | :--- |
| **2xx (e.g., 200 OK)** | Healthy | Subscription remains active. |
| **410 Gone** | Stale/Terminated | The Hub should remove the subscription immediately. |
| **Other (4xx/5xx)** | Unavailable | The Hub may retry later or mark the subscriber as temporarily unreachable. |
| **Timeout/Connection Error** | Unreachable | The Hub may retry or eventually mark as inactive. |

### Subscriber Service Implementation

To ensure seamless adoption, the `websub:SubscriberService` will be updated to provide a default implementation for the `onPing` remote function. This allows existing and new subscribers to support the ping semantic without manual intervention, unless custom health-check logic is required.

```ballerina
public type SubscriberService service object {
    // ... existing remote functions

    # Default implementation for handling ping requests from the Hub.
    # 
    # + topic - The topic for which the ping is sent
    # + return - Returns `websub:Acknowledgement` if healthy, `websub:SubscriptionDeletedError` for HTTP 410, or `error` for other failures
    remote function onPing(string topic) returns websub:Acknowledgement|websub:SubscriptionDeletedError|error? {
        return {
            status: 200,
            body: "OK"
        };
    }
}
```

## Testing

### Unit Testing
- Test `HubClient.ping()` with a mock HTTP server.
- Verify that `hub.mode=ping` is correctly sent as a query parameter.
- Verify that the function returns `()` for 200 OK and returns a `SubscriptionError` for 410 Gone or other error codes.

### Integration Testing
- Use a sample WebSub Subscriber that handles the `hub.mode=ping` request.
- Simulate different scenarios:
    - Subscriber returns 200 OK.
    - Subscriber returns 410 Gone.
    - Subscriber is down (Connection refused).

## Risks and Assumptions

- **Assumption**: Subscribers need to be updated to handle the `hub.mode=ping` request. If they don't, they might return 404 or 405, which the Hub should interpret cautiously (perhaps as "Unavailable" rather than "Gone").
- **Risk**: Frequent pinging can introduce unnecessary network traffic and load on both the Hub and Subscribers. It is recommended that Hubs implement reasonable intervals for health checks.

## Future Work

- Standardizing the `hub.topic` parameter in the ping request for more granular health checks.
