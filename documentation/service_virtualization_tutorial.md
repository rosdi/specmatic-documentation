---
layout: default
title: Service Virtualization
parent: Documentation
nav_order: 6
---
Service Virtualization
======================

- [Service Virtualization](#service-virtualization)
  - [Pre-requisites](#pre-requisites)
  - [Stub Out The Product Service](#stub-out-the-product-service)
  - [Fix The Response To /products/10](#fix-the-response-to-products10)
  - [Add A Second Expectation](#add-a-second-expectation)
  - [Generate The SKU](#generate-the-sku)
  - [Try Setting Up An Invalid Expectation](#try-setting-up-an-invalid-expectation)
  - [Stubbing Out The Request](#stubbing-out-the-request)
  - [Accept Any Value For "name" and "sku" And Return A Random Id](#accept-any-value-for-name-and-sku-and-return-a-random-id)
  - [Generating the stub files](#generating-the-stub-files)
  - [SSL / Https  stubbing](#ssl--https--stubbing)
    - [Autogenerated Cert Store](#autogenerated-cert-store)
    - [Bring your own key store](#bring-your-own-key-store)
  - [Delay Simulation](#delay-simulation)
  - [Dynamic Mocking - Setting expecations over Specmatic Http API](#dynamic-mocking---setting-expecations-over-specmatic-http-api)
    - [Context](#context)
    - [Expectations Http Endpoint](#expectations-http-endpoint)
    - [Anatomy of a Component / API Test](#anatomy-of-a-component--api-test)
  - [Programmatic Approach](#programmatically-starting-stub-server-within-tests)
  - [Sample Java Project](#sample-java-project)

## Pre-requisites

Let's prepare to try out service virtaulization step-by-step.

- [Download the latest version of Specmatic](https://specmatic.in/download/latest.html).
- Create a file named products-api.yaml on your laptop with the following contents.

```yaml
openapi: 3.0.0
info:
  title: Sample Product API
  description: Optional multiline or single-line description in [CommonMark](http://commonmark.org/help/) or HTML.
  version: 0.1.9
servers:
  - url: http://localhost:8080
    description: Local
paths:
  /products/{id}:
    get:
      summary: Get Products
      description: Get Products
      parameters:
        - in: path
          name: id
          schema:
            type: number
          required: true
          description: Numerical Product Id
      responses:
        '200':
          description: Returns Product With Id
          content:
            application/json:
              schema:
                type: object
                required:
                  - name
                  - sku
                properties:
                  name:
                    type: string
                  sku:
                    type: string
  /products:
    post:
      summary: Add Product
      description: Add Product
      requestBody:
        content:
          application/json:
            schema:
              type: object
              required:
                - name
                - sku
              properties:
                name:
                  type: string
                sku:
                  type: string
      responses:
        '200':
          description: Returns Product With Id
          content:
            application/json:
              schema:
                type: object
                required:
                  - id
                properties:
                  id:
                    type: integer

```

- Create a directory named products-api_data in which we will place these stubs.

## Stub Out The Product Service

* Run `java -jar specmatic.jar product-api.yaml`
* Use postman to make a request to http://localhost:9000/products/10

Specmatic returns a contract valid response. The values are randomly generated. We have not yet told Specmatic how to handle this request.

## Fix The Response To /products/10

* Create a file named expectation.json, inside products-api_data, with the following contents:

```json
{
  "http-request": {
    "method": "GET",
    "path": "/products/10"
  },
  "http-response": {
    "status": 200,
    "body": {
      "name": "Soap",
      "sku": "abc123"
    }
  }
}
```

* Wait a few seconds for Specmatic to load the file.
* Make the above request again from Postman.

## Add A Second Expectation

* Rename the above json file to soap.expectation
* Create a new json file named batteries.json in the same directory with the following data:

```json
{
  "http-request": {
    "method": "GET",
    "path": "/products/20"
  },
  "http-response": {
    "status": 200,
    "body": {
      "name": "Batteries",
      "sku": "abc123"
    }
  }
}
```

* Make a request to http://localhost:9000/products/20.

## Generate The SKU

We don't care at the moment what the SKU is, but the one for Batteries is the same as the one for SOAP, and they should be different. Let's fix that.

* In soap.json, put the value "(string)" against "sku" instead of "abc123".
* Do the same in batteries.json.

Make an API call to http://localhost:9000/products/10 and then http://localhost:9000/products/20. Note that their SKUs are now randomly generated, and different.

## Try Setting Up An Invalid Expectation

Let's see what happens when our expectations of the API response do not align with the contract.

* Create a third file named pencils.json, with the following contents.

```json
{
  "http-request": {
    "method": "GET",
    "path": "/products/30"
  },
  "http-response": {
    "status": 200,
    "body": {
      "name": "Pencils",
      "sku": 12345
    }
  }
}
```

* Wait for Specmatic to pick up the new file.

Note how Specmatic highlights
* The path to the error
* The reason for the error.

## Stubbing Out The Request

* Create a new file named book.json

```json
{
  "http-request": {
    "method": "POST",
    "path": "/products",
    "body": {
      "name": "Book",
      "sku": "pqr456"
    }
  },
  "http-response": {
    "status": 200,
    "body": {
      "id": 40
    }
  }
}
```

* Make a POST API call to http://localhost:9000/products, with the JSON body {"name": "Book", "sku": "pqr456"}

You should get a 200 response with a json body containing "id" 40.

## Accept Any Value For "name" and "sku" And Return A Random Id

* Create a new file named any.json

```json
{
  "http-request": {
    "method": "POST",
    "path": "/products",
    "body": {
      "name": "(string)",
      "sku": "(string)"
    }
  },
  "http-response": {
    "status": 200,
    "body": {
      "id": "(number)"
    }
  }
}
```

* Make a POST API call to http://localhost:9000/products, with the JSON body {"name": "any name here", "sku": "any sku value here"}

You should get a 200 response with a JSON body containing a randomized "id".

## Generating the stub files

Authoring the expecations / stub jsons by hand may be hard / time consuming when the API operation has a large response. Instead you can run the stub server with below option to log the request and response data sample which you can then leverage as a starting point to create custom expectations.

```shell
java -jar specmatic.jar stub products-api.yaml --jsonConsoleLog
```

Send a request to this server.

```shell
curl http://localhost:9000/products/1
```

And you can now see the request and response logged in the console where Specmatic stub server is running.

```shell
{
    #...
    "http-request": {
        "path": "/products/1",
        "method": "GET",
        "headers": {
            "Host": "localhost:9000",
            "User-Agent": "curl/7.84.0",
            "Accept": "*/*"
        },
        "body": ""
    },
    "http-response": {
        "status": 200,
        "body": {
            "name": "HCCPY",
            "sku": "CYCOA"
        },
        "status-text": "OK",
        "headers": {
            "X-Specmatic-Result": "success",
            "Content-Type": "application/json",
            "X-Specmatic-Type": "random"
        }
    },
    #...
}
```

This log contains the entire information about the request and response. We just need to copy the "http-request" and "http-response", remove non-mandatory headers etc. and modify it as per our needs.  

## SSL / Https  stubbing

There are multiple ways to run the Specmatic Stub with SSL.

### Autogenerated Cert Store

This is the quickest approach.

```shell
java -jar specmatic.jar stub --httpsKeyStoreDir=<directory to create keysore> --port=443 product-api.yaml
```

This will create a `specmatic.jks` file in the dir that you mentioned above and you can now access the stub over https.

### Bring your own key store

If you already have a keystore and self signed certificate you can pass it to Specmatic through below command options.

```shell
% specmatic stub --help
...
      --httpsKeyStore=<keyStoreFile>
                             Run the proxy on https using a key in this store
      --httpsKeyStorePassword=<keyStorePassword>
                             Run the proxy on https, password for pre-existing
                               key store
      --httpsPassword=<keyPassword>
                             Key password if any
```

## Delay Simulation

Specmatic allows granular control over simulating a slow response for certain requests.

Let us create another expecation file on the same lines as [expectation.json](/documentation/service_virtualization_tutorial.html#fix-the-response-to-products10) in the products-api_data folder and call it expectation-with-delay.json with below content.

```yaml
{
  "http-request": {
    "method": "GET",
    "path": "/products/11"
  },
  "http-response": {
    "status": 200,
    "body": {
      "name": "Slow Soap",
      "sku": "slow1234"
    }
  },
  "delay-in-seconds": 3
}
```

We have set the delay to 3 seconds here. Once the Specmatic stub server has loaded this expectation, time the request for product id 11 and you should see a 3 second delay.

```shell
% time curl http://localhost:9000/products/11
{
    "name": "Slow Soap",
    "sku": "slow1234"
}curl http://localhost:9000/products/11  0.01s user 0.01s system 0% cpu 3.082 total
```

All other requests, other than the specific request (product id 11) where a delay has been setup, will continue to behave as usual.

## Dynamic Mocking - Setting expecations over Specmatic Http API

### Context
It is not always possible to know ahead of time what expecation data needs to be setup. Example: Consider below scenario
* Let us say our system under test needs to lookup the SKU value from another service (over which we do not have control or cannot mock) before creating products
* In this scenario we will be unable to create the expectation json file with any specific value of SKU since we will only know this at test runtime / dynamically

**Dynamic Mocks** are helpful in such scenarios which involve a **work flow** with multiple steps where the input of a step depends on output of its previous step.

### Expectations Http Endpoint

Specmatic stub server can accept expectation json through below http endpoint.

```shell
http://localhost:9000/_specmatic/expectations
```

Please see <a href="/documentation/SpecmaticExpectations-postman_collection.json" download>postman collection</a> for reference.

Specmatic will verify these expecations against the OpenAPI Specifications and will only return a 2xx response if they are as per API Specifications. Specmatic returns 4xx reponse if the expectation json is not as per the OpenAPI Specifications.

### Anatomy of a Component / API Test

<img alt="Anatomy of a Component / API Test" src="https://specmatic.in/wp-content/uploads/2022/09/Contact-as-stub.png" />

Please see this [video](https://youtu.be/U5Agz-mvYIU?t=998) for reference.

The above image shows how Specmatic Smart Mocking fits into your Component Test. A good component test isolates the system / component under test from its dependencies. Here Specmatic is emulating the dependencies of the mobile application thereby isolating it.

**API Tests are just Component Tests where the System Under Test is a Service / API**. Here is an [example](https://github.com/znsio/specmatic-order-ui/blob/main/src/test/kotlin/controllers/apiTests.feature) of how you can leverage Specmatic dynamic mocking in a Karate API Test. Below are the pieces involved.
* **Test** - A Karate API Test
* **System Under Test** - [Find Available Products Service](https://github.com/znsio/specmatic-order-ui/blob/main/src/main/kotlin/controllers/Products.kt) - Invokes products API to get all products and filters out products where inventory is zero.
* **Dependency** - Products API mocked by Specmatic. Specmatic is setup to leverage [OpenAPI Specification of Products API](https://github.com/znsio/specmatic-order-contracts/blob/main/in/specmatic/examples/store/api_order_v1.yaml) in the [central contract repo](https://github.com/znsio/specmatic-order-contracts) through [specmatic.json](https://github.com/znsio/specmatic-order-ui/blob/main/specmatic.json) configuration.

Let us analyse each phase of this API test.
* **Arrange** - In this step we setup Specmatic stub server with expectation json through Specmatic http endpoint to emulate the Products API. We set it up to return two products, a laptop (which is available) and a phone (inventory is zero). We also verify that Specmatic has accepted this expectation data by asserting that the response code is 2xx. This confirms that are our expectation data is in line with the OpenAPI Specification of Products OpenAPI Specification.
* **Act** - Here the Karate test invokes System / Service Under Test (Find Products Service) to exercise the functionality we need to test. This inturn results in the System / Service Under Test invoking its dependency (Products Service) which is being emulated by Specmatic. Specmatic returns the response we have setup in the previous step to the System Under Test. System Under Test processes this data and responds to API Test.
* **Assert** - We now verify the response from System / Service Under Test to ascertain if it has only returned the laptop, since the phone is not available.

The same approach can be leveraged in other test tools and frameworks such as [Rest-Assured](https://rest-assured.io/) also.

## Programmatically starting stub server within tests

If your tests are written in a JVM based language, you can start and stop the stub server within your tests programmatically.

Add `specmatic-core` jar dependency with scope set to test since this need not be shipped as part of your production deliverable.

```
<dependency>
    <groupId>in.specmatic</groupId>
    <artifactId>specmatic-core</artifactId>
    <version>{{ site.latest_release }}</version>
    <scope>test</scope>
</dependency>
```

Now you can import the utilty to create the stub server. Below code snippets are in Kotlin. However the overall concept is the same across all JVM languages such as Clojure, Scala or plain Java.

```kotlin
import `in`.specmatic.stub.createStub
```

This utility can now be used in your test ```setup``` / ```beforeAll``` method to start the stub server. Specmatic automatically looks for your [```specmatic.json```](/documentation/specmatic_json.html) file in project root directory / classpath to locate your API Specification files that need to run as part of the stub server.

```kotlin
@BeforeAll
@JvmStatic
fun setUp() {
    stub = createStub()
}
```

And subsequently once your tests are done, you can shutdown the stub server as part of your ```teardown``` / ```afterAll``` method.

```kotlin
@AfterAll
@JvmStatic
fun tearDown() {
    service?.stop()
    stub.close()
}
```

Here is a complete [example](https://github.com/znsio/specmatic-order-ui/blob/683f59f5024e02af88fb54a55f03d819f852bb2e/src/test/kotlin/controllers/APITests.kt) of a Karate API test that leverages the above technique.

Please note that this is only a utility for the purpose of convenience in Java projects. Other programming languages can simply run the Specmatic standalone executable just as easily. If you do happpen to write a thin wrapper and would like to contribute the same to the project, please refer to our [contribution guidelines](https://github.com/znsio/specmatic/blob/main/CONTRIBUTING.md).

## Sample Java Project

[https://github.com/znsio/specmatic-order-ui](https://github.com/znsio/specmatic-order-ui)

