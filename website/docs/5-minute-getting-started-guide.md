---
title: 5 minute guide
description: From zero to running Pact tests in 5 mins
---


This getting started guide runs purely in the browser, with the intention to get you across the key concepts quickly. As we go through, there will be code snippets to demonstrate concepts. These are not runnable, but are there to assist with explanation. When there is code to run, you will be presented with a REPL as per below. Simply hit the green ▶ button, and the output will be displayed in the terminal below. Give it a shot now:

<iframe frameborder="0" width="100%" height="500px" src="https://repl.it/@mefellows/Getting-Started-Intro?lite=true"></iframe>


## An example scenario: Order API

Here we have an example describing Pact tests between a consumer \(Order Web\), and its provider \(the Order API\).

In the Consumer project, we're going to need:

* A model \(the `Order` class\) to represent the data returned from the Order API
* A client \(the `OrderApiClient`\) which will be responsible for making the HTTP calls to the Order API and returning an internal representation of an Order.

Note that to create a pact, you _do_ need to write the code that executes the HTTP requests to your service \(in your client class\), but you _don't_ need to write the full stack of consumer code \(eg. the UI\).

## Testing the Order Web \(consumer\) project

### Scope of a Consumer Pact Test

Ideally, the Pact tests should be "unit tests" for your client class, and they should just focus on ensuring that the request creation and response handling are correct. If you use pact for your UI tests, you'll end up with an explosion of redundant interactions that will make the verification process tedious. Remember that pact is for testing the contract used for communication, and not for testing particular UI behaviour or business logic.

Usually, your application will be broken down into a number of sub-components, depending on what type of application your consumer is \(e.g. a Web application or another API\). This is how you might visualise the coverage of a consumer Pact test:

![Scope of a consumer Pact test](/img/consumer-test-coverage.png)

Here, a _Collaborator_ is a component whose job is to communicate with another system. In our case, this is the `OrderApiClient`communicating with the external `Order Api` system. This is what we want our consumer test to inspect.

### 1. Start with your model

Imagine a simple model class that looks something like this \(order.js\). The attributes for an Order live on a remote server, and will need to be retrieved by an HTTP call to the Order API.

<!--DOCUSAURUS_CODE_TABS-->
<!--order.js-->

```js
class Order {
  constructor(id, items) {
    this.id = id
    this.items = items
  }

  total() {
    return this.items.reduce((acc, v) => {
      acc += v.quantity * v.value
      return acc
    }, 0)
  }

  toString() {
    return `Order ${this.id}, Total: ${this.total()}`
  }
}
```

<!--Sample Order-->

```js
module.exports = [
  {
    id: 1,
    items: [
      {
        name: 'burger',
        quantity: 2,
        value: 20,
      },
      {
        name: 'coke',
        quantity: 2,
        value: 5,
      },
    ],
  },
]
```
<!--END_DOCUSAURUS_CODE_TABS-->

### 2. Create an Order API client

Here we have our external collaborator client. Its job is to both make the external request to the Order API and convert the response into the internal Order model as per above:


<!--DOCUSAURUS_CODE_TABS-->
<!--orderClient.js-->

```js
const fetchOrders = () => {
  return request.get(`${API_ENDPOINT}/orders`).then(
    res => {
      return res.body.map((o) => {
        return new Order(o.id, o.items)
      })
    },
    err => {
      throw new Error(`Error from response: ${err.body}`)
    }
  )
}
```

<!--END_DOCUSAURUS_CODE_TABS-->

### 3. Configure the mock Order API

The following code will create a mock service on `localhost:1234` which will respond to your application's queries over HTTP as if it were the real Order API. It also creates a mock provider object which you will use to set up your expectations.

```js
// Setup Pact
const provider = new Pact({
  port: 1234,
  log: path.resolve(process.cwd(), "logs", "pact.log"),
  dir: path.resolve(process.cwd(), "pacts"),
  consumer: "OrderWeb",
  provider: "OrderApi"
});

// Start the mock service!
await provider.setup()
```

### 4. Write a test

<!--DOCUSAURUS_CODE_TABS-->
<!-- order.spec.js -->
```js
describe('Pact with Order API', () => {
  describe('given there are orders', () => {
    describe('when a call to the API is made', () => {
      before(() => {
        return provider.addInteraction({
          state: 'there are orders',
          uponReceiving: 'a request for orders',
          withRequest: {
            path: '/orders',
            method: 'GET',
          },
          willRespondWith: {
            body: eachLike({
              id: 1,
              items: eachLike({
                name: 'burger',
                quantity: 2,
                value: 100,
              }),
            }),
            status: 200,
            headers: {
              'Content-Type': 'application/json; charset=utf-8',
            },
          },
        })
      })

      it('will receive the list of current orders', () => {
        return expect(fetchOrders()).to.eventually.have.deep.members([
          new Order(orderProperties.id, [itemProperties]),
        ])
      })
    })
  })
})
```
<!--END_DOCUSAURUS_CODE_TABS-->

Green!

Running the passing Order API spec will generate a pact file in the configured pact dir \(`./pacts` by default\). Logs will be output to the configured log dir \(`./log` by default\) that can be useful when diagnosing problems.

You now have a pact file that can be used to verify your expectations of the Order API provider project.

Now, in real life you would rinse and repeat for other likely status codes that may be returned. For example, consider how you want your client to respond to a:

* 404 \(return null, or raise an error?\)
* 400 \(how should validation errors be handled, what will the body look like when there is one?\)
* 500 \(specifying that the response body should contain an error message, and ensuring that your client logs that error message will make your life much easier when things go wrong. Note that it may be hard to force your provider to generate a 500 error on demand. You may need to collaborate with your provider team to create a known provider state that will artificially return a 500 error, or you may just wish to use a standard unit test without a pact to test this.\)
* 401/403 if there is authorisation.

#### **Run the consumer Tests!**

OK enough talk - let's run the consumer test. If you like, click around the project to see the files from above in context. The most interesting file is the consumer test in `./consumer/consumer.spec.js` .

<iframe frameborder="0" width="100%" height="500px" src="https://repl.it/@mefellows/docspactio-getting-started?lite=true"></iframe>

## Sharing the contracts with the provider team

Now that you have created and run your consumer tests, producing a contract \(the pact file\) as an artefact, we need to share it with the team responsible for managing the Order API, so that they can confirm they meet all of the expectations set in it. There are multiple ways to [share pacts](getting_started/sharing_pacts.md), but the recommended approach is to use a [Pact Broker](pact_broker/) as it enables powerful automation workflows.

For this purpose, we are going to use a hosted Pact Broker from pactflow.io.

> The Pact Broker is an open source tool that requires you to deploy, administer and host it yourself. If you would prefer a plug-and-play option, we've created [Pactflow](https://pactflow.io/?utm_source=ossdocs&utm_campaign=five_minute_guide), a fully managed Pact Broker with additional features to simplify teams getting started and scaling with Pact.
>
> To get started for free, you can sign up to our Developer Plan [here](https://pactflow.io/pricing/?utm_source=ossdocs&utm_campaign=five_minute_guide_dev_plan).


<!-- <iframe style="padding-bottom:20px" frameborder="0" width="100%" height="500px" src="https://repl.it/@mefellows/docspactio-getting-started-publish?lite=true"></iframe> -->

You can see the published pact [here](https://test.pact.dius.com.au/pacts/provider/GettingStartedOrderApi/consumer/GettingStartedOrderWeb/latest). The Pactflow account is protected using basic auth. Use the username `dXfltyFMgNOFZAxr8io9wJ37iUpY42M`, and password `O5AIZWxelWbLvqMd8PkAVycBJh2Psyg1`.

After publishing the pact, we can now verify that the Provider meets these expectations.

## Testing the Order API \(provider\) project

### Scope of a Provider Pact Test

On the Provider side, Pact needs to replay all of the interactions \(usually HTTP requests\) against your service. There are a number of choices that can be made here, but usually these are the choices:

* Invoke just the controller layer \(in an MVC app, or the "Adapter" in our diagram\) and stub out layers beneath
* Choosing a real vs mocked out database
* Choosing to hit mock HTTP servers or mocks for external services

Generally speaking, we test the entire service and mock out external services such as downstream APIs \(which would need their own set of Pact tests\) and databases. This gives you some of the benefits of an integration test without the high costs of maintenance.

This is how you might visualise the coverage of a provider Pact test:

![Provider side Pact test scope](/img/provider-test-coverage.png)

### 1. Create the Order API

Below we have created a simple API using [Express JS](https://expressjs.com).

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

<Tabs
  groupId="sdk-choice"
  defaultValue="javascript"
  values={[
    { label: 'Javascript', value: 'javascript', },
  ]
}>
  <TabItem value="javascript">

  ```js
  const express = require('express')
  const cors = require('cors')
  const bodyParser = require('body-parser')
  const server = express()

  server.use(cors())
  server.use(bodyParser.json())
  server.use(bodyParser.urlencoded({ extended: true }))
  server.use((_, res, next) => {
    res.header('Content-Type', 'application/json; charset=utf-8')
    next()
  })

  // "In memory" data store
  let dataStore = require('./data/orders.js')

  server.get('/orders', (_, res) => {
    res.json(dataStore)
  })

  module.exports = {
    server,
    dataStore,
  }
  ```

  </TabItem>
</Tabs>

### 2. Run provider verification tests

We now need to perform the "provider verification" task, which involves the following:

1. Telling Pact where to find the contract files, and where the Order API will be running \(lines 3-13\)
2. Starting the API \(line 16-18\)
3. Running the Provider verification task \(line 22\)


<Tabs
  groupId="sdk-choice"
  defaultValue="javascript"
  values={[
    { label: 'Javascript', value: 'javascript', },
  ]
}>
  <TabItem value="javascript">

  ```js
// Verify that the provider meets all consumer expectations
  describe('Pact Verification', () => {
    const port = 1234
    const opts = {
      provider: providerName,
      providerBaseUrl: `http://localhost:${port}`,
      pactBrokerUrl: 'https://test.pact.dius.com.au/',
      pactBrokerUsername: 'dXfltyFMgNOFZAxr8io9wJ37iUpY42M',
      pactBrokerPassword: 'O5AIZWxelWbLvqMd8PkAVycBJh2Psyg1',
      publishVerificationResult: true,
      tags: ['prod'],
      providerVersion: '1.0.' + process.env.HOSTNAME,
    }

    before(async () => {
      server.listen(port, () => {
        console.log(`Provider service listening on http://localhost:${port}`)
      })
    })

    it('should validate the expectations of Order Web', () => {
      return new Verifier().verifyProvider(opts)
    })
  })
  ```

  </TabItem>
</Tabs>


#### Run the Provider tests

Let's run the provider test. If you like, click around the project to see the files from above in context. The most interesting file is the consumer test in `./provider/provider.spec.js` .

<iframe frameborder="0" width="100%" height="500px" src="https://repl.it/@mefellows/docspactio-getting-started-provider?lite=true"></iframe>

