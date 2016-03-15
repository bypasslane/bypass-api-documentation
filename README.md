# Introduction

Welcome to the Bypass API!

This documentation is provided as a guide for developing integrations to consume Bypass data as well as developing applications on top of Bypass APIs. Specifically, in this guide we will cover the the guest checkout sequence for in-seat delivery to a customer in a stadium environment.

# Architecture

## Data Interaction

The Bypass API leverages a REST architecture with JSON as the format for data exchange.
Most resource endpoints implement standard resource CRUD (Create / POST, Read / GET, Update / PUT, Delete / DELETE) methods, though some are reserved.

API endpoint examples include (all under https://api.bypassmobile.com/):

* ```GET /api/venue/orders``` to get a list of orders for a venue
* ```GET /api/venue/location/:location_id/orders``` to get a list of orders for a location within the venue
* ```POST /api/venue/location/:location_id/orders``` to create an order at a location within the venue
* ```PUT /api/venue/orders/:order_uuid/payments/:payment_uuid``` to add a payment to an existing order

## Security

Security is a primary concern of the Bypass API.
All API calls use HTTPS and most use an API session token to authenticate the user.
Session tokens are obtained from the Bypass authentication service (https://auth.bypassmobile.com).
Users, roles, and permissions are configured via the Bypass admin (https://admin.bypassmobile.com).

As a POS service provider, extra care is given to the handling of credit card data.
Credit cards swiped on Bypass POS terminals are encrypted at the magnetic stripe read head so that no card holder data is accessible to any service not explicitly designated for PCI compliance.
For the handling of manually entered credit card numbers, the data is sent directly to a dedicated, PCI-compliant service (https://zuul.bypassmobile.com).

## Distributed Systems

The Bypass API is built as a distributed system.
In summary this means that resources available within the API can be created or modified throughout the network and the changes will propagate out to the rest of the system as needed.
To accomplish this, resources designed for distributed management are tagged with a client-generated UUID upon creation and that UUID serves as the global identifier of the resource across all services.

An example of this approach can be seen when a client POS application tells the Bypass server that a payment has been attached to an existing order:

```
PUT https://api.bypassmobile.com/api/venue/orders/2832c9df-c8f9-400c-a14b-6e0c0db1c07d/payments/c75eb4eb-ba8b-4045-91f4-1824f70b8e5a

{
	payment: {
		amount: "10.00",
		payment_type: "cash",
		uuid: c75eb4eb-ba8b-4045-91f4-1824f70b8e5a
	}
}

```

This approach also lends itself to the development of idempotent APIs.
Specifically when dealing with poor network reliability in high traffic sports and entertainment venues, network failure is more than an edge-case. It is assumed that network calls will fail frequently and therefore it is critical that any call can be retried without concern for duplicated data or unexpected behavior.

## Scalability

Bypass achieves scalability via our distributed SOA for resource management.
Distributed resource creation and eventual consistency for data propagation permit the horizontal and vertical scaling of independent services based on the changing needs of the system.

This approach enables two key elements of the API:

1. Consistently fast writes for orders without the need for server-side order math driven by config lookups
2. Background processing of redundant data to offload longer tasks and retain backups of data for recovery scenarios

# The guest checkout sequence

A simple POS application offering guest checkout can be created using the Bypass API via the following flow:

1. Authenticate the user / application
2. Fetch venue data
3. Fetch menu for desired location
4. Create Order locally
5. Authorize CC payment
6. Send order to Bypass

## Authenticate the user / application

In the case of a guest checkout application it can be assumed that the same user will place every order.
As such a single Order Taker can be created for the application within the Bypass admin application (the process for accomplishing this task is not covered in this documentation).

Once an Order Taker is obtained, a session token for the application can be obtained from the Bypass auth service using the following example code:

```javascript
var request = require('request-promise');

var authToken = new Buffer(
  "example_user_name" + ":" + "example_password"
).toString('base64');

var options = {
  url: 'https://auth.bypassmobile.com/auth.json',
  method: 'POST',
  headers: {
    'Accept': 'application/json, text/javascript, */*; q=0.01',
    'Authorization': 'Basic ' + authToken
  },
  json: true
};

request(options).then(function(session) {
  var sessionToken = session.session_token;
  var venueId = session.venue_id;
});
```

## Fetch Venue data

In our example, the venue to be used by the application is determined by the Order Taker used for authentication since each Bypass Order Taker can belong to only one Venue.
As such, the data for the Venue can be obtained from the Bypass API via:

```javascript
var request = require('request-promise');

var sessionToken = "example session token";
var venueId = 86; //<-- Example venue ID

var options = {
  url: 'https://api.bypassmobile.com/api/venues/' + venueId,
  method: 'GET',
  headers: {
    'Accept': 'application/json, text/javascript, */*; q=0.01',
    'X_SESSION_TOKEN': sessionToken
  },
  transform: function(response) {
  	return response["venue"];
  },
  json: true
};

request(options).then(function(venue) {
	// Handle response
});
```

An example Venue response would include (subset of all fields shown below):

```javascript
{ id: 86,
  name: 'Bypass WORLD Headquarters',
  concessions:
  	[{ id: 319,
	   name: 'Example location',
       description: 'Example description',
       has_inseat: true,
       has_pickup: true,
       is_merchandise: false,
       status: 'open'
	}]
}
```

## Fetch Menu for desired location

Once a location for the order has been selected from the list provided for the Venue, it is necessary to fetch the menu for that location. This can be performed via a simple HTTP GET request:

```javascript
var rp = require('request-promise');

var concessionId = 319; //<-- Example location ID

var options = {
  url: 'https://goliath.bypassmobile.com/locations/' + concessionId + '/menu.json',
  method: 'GET',
  json: true
};

rp(options).then(function(menu) {
	// Handle response
});
```

An example Menu response would include (subset of all fields shown below):

```javascript
{ categories:
    [{ id: 442,
       name: 'Food',
       items:
         [{ id: 9249,
            name: 'Buffalo Chicken Sandiwch',
            price: '10.0',
            alcohol: false
         }]
    }]
}

```

The downloaded menu can now be displayed to the user.

## Create Order locally

When a user adds items to their order cart within the POS application, the menu items must be converted into line items.
As discussed previously, this process of creating a POS order is distributed and does not depend on any server-side interaction.
The POS client is responsible for creating an order object that properly represents the customer's order.

A simple order will look something like (subset of all fields shown below):

```javascript
{ order:
	{ uuid: "6a1d10e0-dcab-4de3-979a-4cdb2e0ee428",
		state: "open",
		total: "10.00",
		balance_due: "10.00",
		location_id: 319,
		line_items:
			[{ uuid: "200f0dd7-2998-4488-af9e-630eb8bf132b",
				name: "Buffalo Chicken Sandiwch",
				price: "10.00",
				count: 1
			}],
		payments: []
	}
}
```

Note that the order above is open and has no payments associated with it.
Additionally, it can be seen that UUIDs have been generated and placed on both the order and each of the line items since the POS client is responsible for creating these objects.

## Authorize CC Payment

Now that an order has been constructed, a payment can be created and attached to the order. In this example a credit card payment will be used.

To authorize a CC payment, the application should construct a JSON payload of the necessary data and submit to the Bypass authorization service (https://zuul.bypassmobile.com).

```javascript
var rp = require('request-promise');

var transactionRequest = {
  uuid: "fccb00f0-ac36-4672-a68f-9da5c86393c8",
  transaction: {
    uuid: "fccb00f0-ac36-4672-a68f-9da5c86393c8",
    gateway_account_login: 'example gateway account login',
    amount: 1000, // all amounts on authorization service are in pennies
    credit_card: {
      number: "CC account number. Unencrypted is OK here",
      first_name: "John",
      last_name: "Doe",
      year: 2017,
      month: 10,
      encrypted: false
    }
  }
};

var options = {
  url: 'http://zuul.bypassmobile.com/sales/fccb00f0-ac36-4672-a68f-9da5c86393c8',
  method: 'PUT',
  body: transactionRequest,
  json: true
};

rp(options).then(function(transaction) {
    // Handle response
});
```

The payment attached to the order references the transaction sent to the the authorization service by using the same UUID.

```javascript
{ order:
	{ uuid: "6a1d10e0-dcab-4de3-979a-4cdb2e0ee428",
		state: "closed",
		total: "10.00",
		balance_due: "0.00",
		location_id: 319,
		line_items:
			[{ uuid: "200f0dd7-2998-4488-af9e-630eb8bf132b",
				name: "Buffalo Chicken Sandiwch",
				price: "10.00",
				count: 1
			}],
		payments:
			[{ uuid: "fccb00f0-ac36-4672-a68f-9da5c86393c8",
				amount: "10.00",
				payment_type: "credit"
			}]
	}
}
```

Note that the order state is now "closed" and the balance due is $0.00.

## Send order to Bypass

Now that the order has been fully constructed and payment authorized, the order can be sent to Bypass for fulfillment. This can be performed via an HTTP POST:

```javascript
var rp = require('request-promise');

var order = {} // See definition of completed order from previous section

var options = {
  url: 'http://api.bypassmobile.com/api/venue/concessions/319/orders',
  method: 'POST',
  body: order,
  json: true
};

rp(options).then(function(order) {
    // Handle response
});
```

The order will now be recorded in the Bypass server-side database, sent for fulfillment as needed, updated in perpetual inventory tracking, included in all reports, etc. etc.

# Additional POS order actions

Once an order is recorded in the Bypass API additional actions can be taken on that order.
Some of these include:

* ```POST /api/venue/orders/:order_uuid/email``` to send an email receipt for the order
* ```POST /api/venue/orders/:order_uuid/refund``` to refund a closed order
* ```POST /api/venue/orders/:order_uuid/void``` to void an open order
