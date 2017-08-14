# Azure Resource Manager Resource Provider Contract (RPC)

## Abstract
This document covers the API contract that must be implemented by each Azure Resource Provider in order to onboard to the Azure management API surface (as well as RBAC, tags, and templates).

The document is divided into five major sections. The first section covers the request and respose details common to all APIs. It includes information around the headers, client timeout, throttling, response size limitations etc. The second section covers the details of APIs around subscription lifecycle management for a particular resource provider. The next section covers the create, update, delete and read APIs that needs to be implemented for the resources exposed by the provider. The fourth section covers APIs or resources which will not be managed by Azure Resource Manager (ARM) but will be proxied directly to the resource provider. The last section covers details on various things like asynchronous operations, tracing requests, guidance on designing resources and so on.

The intended audience of this document is any service who wants to onboard to Azure as a resource provider.

## Table of Contents
1. [Common API Details](v1.0/common-api-details.md) <br/>
2. [Subscription Lifecycle API Reference](v1.0/subscription-lifecycle-api-reference.md) <br/>
3. [Resource API Reference](v1.0/resource-api-reference.md) <br/>
4. [Proxy API Reference](v1.0/proxy-api-reference.md)
5. [Addendum](v1.0/Addendum.md)
6. [Breaking change guidelines](v1.0/version-change.md)

## Code of Conduct
This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/). For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or contact opencode@microsoft.com with any additional questions or comments.

## This repository
This repository contains a collection of documents and related materials supporting the overall Microsoft REST API Guidelines initiative. To contribute to this repository, please open issues or send a pull request.
