# Architecture Notes

## Overview

The stack is split into four CDK nested stacks — NetworkingStack, StorageStack, ComputeStack, and ApiStack — each owning a clear layer of the system [from-code]. S3 (AES-256 encrypted) holds raw documents; an S3 event triggers the DocumentProcessor Lambda inside a private VPC subnet, which writes extracted metadata to DynamoDB with Streams enabled and on-demand billing [from-code]. A separate ApiHandler Lambda sits behind API Gateway with a custom Lambda Authorizer validating API keys before any request touches storage [from-code]. The non-obvious design choice is using Gateway-type VPC endpoints for both S3 and DynamoDB — no Interface endpoint hourly charges, no NAT Gateway traffic costs — while an Interface endpoint covers API Gateway for internal consumers [inferred]. Splitting concerns across four stacks means cross-stack references are wired via CDK Outputs, which forces a specific deployment order and makes stack teardown non-trivial if you ever need to remove just one layer [editorial].

## Key Decisions

- Gateway VPC endpoints for S3 and DynamoDB cost nothing per hour but only work within the same region; any cross-region call falls outside the endpoint and either fails or routes over the public internet, which will bite you if you ever add a DR region [from-code]
- On-demand DynamoDB billing is correct for unpredictable document volumes but becomes 6-7x more expensive than provisioned capacity once throughput stabilises above ~200 WCU sustained — you'll want to revisit this at scale [editorial]
- Separate IAM execution roles per Lambda (DocumentProcessor vs ApiHandler) means clean blast-radius isolation, but it also means two separate role ARNs to track in audit reports and two sets of permission drift to watch in IAM Access Analyzer [from-code]
- Custom Lambda Authorizer for API key validation adds a cold-start latency hit on the first request in a burst; without authorizer result caching tuned correctly, every cold invocation pays the full authorizer execution time before the actual handler runs [inferred]
- DynamoDB Streams feeding a Notification Lambda adds at-least-once delivery semantics — without idempotency logic in that Lambda, a stream shard retry after a partial batch failure will re-process records and potentially double-send status notifications [inferred]