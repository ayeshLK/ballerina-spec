# Notify Subscriber on Internal Hub Errors

- Author: @ayeshLK
- Reviewed by: 
- Created date: 2026-03-10
- Updated date: 2026-03-10
- Issue: [#1440](https://github.com/ballerina-platform/ballerina-spec/issues/1440)
- State: Submitted

## Summary

This proposal introduces a mechanism to notify WebSub subscribers of internal hub errors that occur after the subscription intent has been verified. Specifically, it addresses scenarios where a message broker or persistence layer fails during the final stages of subscription processing, ensuring that the subscriber is informed of the failure and can avoid a state mismatch.

## Motivation

Currently, the WebSub hub verifies a subscriber's intent asynchronously. If a failure occurs after the subscriber's intent is successfully verified (e.g., during persistence to a message broker or database), the hub might fail to record the subscription. However, the subscriber, having received the verification request, assumes the subscription is active. This leads to a state mismatch where the subscriber expects updates that will never arrive because the hub has no record of the subscription.

Providing an asynchronous error notification path allows the hub to inform the subscriber of such internal failures, enabling the subscriber to take corrective action (e.g., retrying the subscription).

## Goals

- Introduce a standard way for WebSub hubs to notify subscribers of internal errors post-intent verification.
- Update the `websubhub` library to support sending these notifications.
- Update the `websub` library to support receiving and handling these notifications.

## Non-Goals

- Changing the core WebSub protocol verification flow.

## Design

The proposed solution involves introducing a new `hub.mode` and updating both the `websubhub` and `websub` libraries.

### 1. `websubhub` Library Enhancements

A new `hub.mode` value, `hub-error`, will be introduced to signal internal hub failures to the subscriber.

When an error occurs in the hub (e.g., within the `onSubscriptionIntentVerified` flow) after the intent has been verified, the hub will perform an asynchronous **HTTP GET** request to the subscriber's callback URL with the following query parameters:

- `hub.mode`: Set to `hub-error`.
- `hub.topic`: The topic URL that the subscriber attempted to subscribe to.
- `hub.reason`: A descriptive error message explaining the failure.

Example request:
`GET https://subscriber.com/callback?hub.mode=hub-error&hub.topic=http://example.com/topic&hub.reason=Broker+unavailable`

### 2. `websub` Library Enhancements

The `websub` library will be updated to handle this new notification mode.

- **New Remote Method:** A new method, `onHubError`, will be added to the `websub:SubscriberService` interface.
  
  ```ballerina
  remote function onHubError(websub:InternalHubError hubError) returns error? {
      // User-defined logic to handle the error
  }
  ```

- **Error type:** A new error type `websub:InternalHubError` will be introduced.

  ```ballerina
  public type InternalHubError distinct Error;
  ```

- **Callback Handling:** The `websub` library's listener will be updated to parse incoming GET requests where `hub.mode=hub-error` and dispatch them to the `onHubError` remote method of the relevant service.

## Alternatives

### Synchronous Persistence
One alternative is to make subscription persistence synchronous during the intent verification phase. However, this was rejected because intent verification should be as lightweight as possible to avoid unnecessary processing and potential timeouts during the verification handshake.

### Subscriber Polling
Another alternative is for subscribers to periodically poll the hub for their subscription status. This was rejected as it contradicts the push-based nature of the WebSub protocol and increases overhead for both the hub and the subscriber.

## Testing

### Unit Tests
- Verify that the `websub` library correctly parses the `hub-error` mode and its parameters from a GET request.
- Verify that the `websubhub` library correctly constructs the error notification request.

### Integration Tests
- Mock a failure in the `onSubscriptionIntentVerified` flow of a Hub.
- Verify that the Hub sends a GET request to the subscriber's callback URL.
- Verify that the subscriber's `onHubError` method is invoked with the expected `hub.topic` and `hub.reason`.

## Risks and Assumptions

- **Assumption:** The subscriber's callback URL is reachable when the hub error occurs.
- **Risk:** If the hub itself crashes before sending the error notification, the state mismatch will still exist. This proposal addresses transient failures in downstream components like brokers or databases.

## Dependencies

- This proposal depends on updates to both `ballerina/websubhub` and `ballerina/websub` modules.
