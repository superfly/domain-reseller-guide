# Domain Reseller Guide

This repo is a living guide on reselling domains with fly for SaaS app builders.

## How It works
Fly is launching APIs for programatic domain registration and DNS management. You can use these APIs to integrate custom domain reselling into your app. Domains are registered by fly (which means free WHOIS privacy for your customers) and can be transferred out anytime. 

## Billing

We are offering registration and renewal at cost. We charge your fly account immediately following a successful registration, transfer, or renewal. It's up to you to set your prices and bill your customers.

## Implementation

The implementation will vary from app to app, but here’s the gist:

**1. build a domain search and registration UI into your app**

The custom domain ux is totally up to you. At a minimum you'll need a page that lets your customers search for a domain and view availability with a button to buy a domain. Behind the scenes, you'll use our api to search and register the domain. 

**2. handle registration**

When a domain is registered you'll want to record it in your database. This will let you manage domains without calling out to our API as well as to map requests for a custom domain to the right customer later on.

**3. configure DNS**

Once the domain is registered you can add the necessary DNS records to point the new domain to your app. The most common setup will be:
- an A record at the zone apex with your fly app's IPv4 address
- an AAAA record at the zone apex with your fly app's IPv6 address
- _optionally_ an A record at `www` with your fly app's IPv4 address
- _optionally_ an AAAA record at `www` with your fly app's IPv6 address

**4. configure SSL certicicate**

Next you'll create an SSL certificate for the new domain. The DNS records from earlier are used for validation.

**5. handle requests to the custom domain**

Once DNS and SSL is configured, requets to the new domain will reach your web app. Lookup the customer by matching the HOST header to the domain's record in your database 

## DNS Management Portal

Inevitability your customers will need to manage custom DNS records for their domain, such as mail servers or Google domain verification records. You can use the DNS management portal api to create minimal branded UIs to let your customers create and modify DNS records themselves. To use it, you'll use our API to generate a signed url which will render the portal for a limited time. 

![Example DNS Portal](/docs/example-portal.png "Example DNS Portal")

To use it, you first need to create a portal using the createDnsPortal mutation. It's helpful to name the portal and lookup on demand as opposed to storing the ID.

To create a session for a customer

1. lookup the domain and portal ids using your organization slug, portal name, and domain name. [Example](#lookup-a-portal-and-domain)
2. create a session using the portal id and domain id [Example](#create-a-dns-portal-session)
3. redirect the customer to the url returned from the createDnsPortalSession mutation.


# Using the GraphQL API

The fly API is built on GraphQL. If you're not familiar, take a look at the [official docs](https://graphql.org).

## Endpoint

The GraphQL api endpoint is available at 

```
https://api.fly.io/graphql
```

## Authentication

You'll use a personal access token to authenticate your fly user to the api. You can create one in the [fly dashboard](https://web.fly.io/user/personal_access_tokens) or get your current one with `flyctl auth token`

## Making Requests

The API will respond to any POST request with a well-formed JSON body. Most languages have clients, but anything that makes http requests will work. Here's an example using curl:

```bash
curl https://api.fly.io/graphql -X POST \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer $(flyctl auth token)' \
  -d ' {
      "query": "query { viewer { id name email } }"
  }'
```

## Playground

Our GraphQL Playground is available at https://api.fly.io/graphql. This is a great tool for exploring the api and constructing queries that you can copy over to your app or scripts.

## Object IDs
Most mutations that operate on an object require the object's [global node id](https://graphql.org/learn/global-object-identification/) as an argument. Use the fields on query to lookup objects.

### Lookup organization by slug

**Query**

```graphql
query {
  organization(slug: "your-org") {
    id
    name
    slug
  }
}
```

**Response**

```json
{
  "data": {
    "organization": {
      "id": "bjkZRb7BXzk36H381gwbX0z9bXRHZvM",
      "name": "Your Org",
      "slug": "your-org"
    }
  }
}
```



### Lookup domain by name

**Query**

```graphql
query {
  domain(name: "example.com") {
    id
    name
  }
}
```

**Response**

```json
{
  "data": {
    "domain": {
      "id": "bjkZRb7BXzk36H381gwbX0z9bXRHZvM",
      "name": "example.com"
    }
  }
}
```

# Examples

## Searching for Domains

The [`checkDomain`](https://api.fly.io/graphql/docs/mutation/checkdomain/) mutation returns domain availability and pricing.

**Query**

```graphql
mutation {
 checkDomain(input: {
   domainName: "fly.io"
 }) {
   domainName
   tld
   registrationSupported
   registrationAvailable
   registrationPrice
   registrationPeriod
   transferAvailable
   dnsAvailable
 }
}
```

**Response**

```json
{
  "data": {
    "checkDomain": {
      "domainName": "fly.dev",
      "tld": "dev",
      "registrationAvailable": false,
      "registrationSupported": true,
      "registrationPrice": 1800,
      "registrationPeriod": 1,
      "dnsAvailable": true
    }
  }
}
```



## Create and Register a Domain

The [`createAndRegisterDomain`](https://api.fly.io/graphql/docs/mutation/createAndRegisterDomain/) mutation creates and registers a new domain in the provided organization. The input requires an organization node id which you can find on an [`Organization`](https://api.fly.io/graphql/docs/object/organization/#id) object.

**Query**

```graphql
  mutation {
    createAndRegisterDomain(input: {
      organizationId: "bjkZRb7BXzk36H381gwbX0z9bXRHZvM",
      name: "example.com"
    }){
      domain {
        id
        name
        registrationStatus
        dnsStatus
        organization {
          id
          slug
        }
      }
    }
  }
```

**Response**

```json
{
  "data": {
    "createAndRegisterDomain": {
      "domain": {
        "id": "vwvgDqn4kxLk58fsSgGAndh8z",
        "name": "example.com",
        "registrationStatus": "REGISTERING",
        "dnsStatus": "pending",
        "organization": {
          "id": "bjkZRb7BXzk36H381gwbX0z9bXRHZvM",
          "slug": "your-org"
        }
      }
    }
  }
}
```

**An error response**

```json
{
  "data": {
    "createAndRegisterDomain": null
  },
  "errors": [
    {
      "message": "Validation failed: Name has already been taken",
      "locations": [
        {
          "line": 2,
          "column": 3
        }
      ],
      "path": [
        "createAndRegisterDomain"
      ],
      "extensions": {
        "code": "UNPROCESSABLE"
      }
    }
  ]
}
```

## Create and Transfer a Domain

The [`createAndTransferDomain`](https://api.fly.io/graphql/docs/mutation/createAndTransferDomain/) mutation creates a new domain in the provided organization and starts a transfer. The input requires an organization node id which you can find on an [`Organization`](https://web.fly.io/graphql/docs/object/organization/#id) object and an authorization code.

**Query**

```graphql
  mutation {
    createAndTransferDomain(input: {
      organizationId: "bjkZRb7BXzk36H381gwbX0z9bXRHZvM",
      name: "example.com",
      authCode: "abc123"
    }){
      domain {
        id
        name
        registrationStatus
        dnsStatus
        organization {
          id
          slug
        }
      }
    }
  }
```

**Response**

```json
{
  "data": {
    "createAndTransferDomain": {
      "domain": {
        "id": "vwvgDqn4kxLk58fsSgGAndh8z",
        "name": "example.com",
        "registrationStatus": "TRANSFERRING",
        "dnsStatus": "pending",
        "organization": {
          "id": "bjkZRb7BXzk36H381gwbX0z9bXRHZvM",
          "slug": "your-org"
        }
      }
    }
  }
}
```

**An error response**

```json
{
  "data": {
    "createAndTransferDomain": null
  },
  "errors": [
    {
      "message": "Validation failed: Name has already been taken",
      "locations": [
        {
          "line": 2,
          "column": 3
        }
      ],
      "path": [
        "createAndTransferDomain"
      ],
      "extensions": {
        "code": "UNPROCESSABLE"
      }
    }
  ]
}
```


## Lookup a Domain by Name

Use the [`domain`](https://api.fly.io/graphql/docs/operation/query/#domain) field to find a domain by name.

**Query**

```graphql
query {
  domain(name: "example.com") {
     id
     name
     registrationStatus
     dnsStatus
  }
}
```

**Response**

```json
{
  "data": {
    "createAndRegisterDomain": {
      "domain": {
        "id": "vwvgDqn4kxLk58fsSgGAndh8z",
        "name": "example.com",
        "registrationStatus": "REGISTERED",
        "dnsStatus": "ready"
      }
    }
  }
}
```

## List all domains

Domains are scoped to an organization. Use the [`domains`](https://api.fly.io/graphql/docs/object/organization/#domains) field on an organization to access it's domains.

**Query**

```graphql
query {
  organization(slug: "your-org") {
    domains(first: 25) {
      totalCount
      nodes {
        id
        name
        registrationStatus
        dnsStatus
      }
    }
  }
}
```

**Response**

```json
{
  "data": {
    "organization": {
      "domains": {
        "totalCount": 1,
        "nodes": [
           {
             "id": "vwvgDqn4kxLk58fsSgGAndh8z",
             "name": "example.com",
             "registrationStatus": "REGISTERED",
             "dnsStatus": "ready"
           }
        ]
      }
    }
  }
}
```

## Add a DNS Record to a Domain

The [`createDnsRecord`](https://api.fly.io/graphql/docs/mutation/creatednsrecord/) mutation creates a DNS record in the provided domain's hosted zone. The input requires a domain's node id which you can find on a [`Domain`](https://api.fly.io/graphql/docs/object/domain/#id) object.

**Query**

```graphql
mutation {
  createDnsRecord(
    input: {
      domainId: "vwvgDqn4kxLk58fsSgGAndh8z"
      type: A
      rdata: "1.2.3.4"
      name: "www"
      ttl: 300
    }
  ) {
    record {
      id
      name
      fqdn
      type
      ttl
      rdata
    }
  }
}

```

**Response**

```json
{
  "data": {
    "createDnsRecord": {
      "record": {
        "id": "P4XYgBwwqqKHwKVmGeaznAgGu6D28nV",
        "name": "www",
        "fqdn": "www.example.com.",
        "type": "A",
        "ttl": 300,
        "rdata": "1.2.3.4"
      }
    }
  }
}
```

## Create a DNS Portal

The [`createDnsPortal`](https://api.fly.io/graphql/docs/mutation/creatednsportal/) mutation creates a DNS portal in the provided organization. The input requires an organization's node id which you can find on an [`Domain`](https://api.fly.io/graphql/docs/object/organization/#id) object. Portals can optionally be named to make lookups easier.

There are several options for customizing the portal:

1. Return URL - use the `returnUrl` and `returnUrlText` input fields to customize the return button. If not provided no return button will be displayed.
2. Support URL - use the `supportUrl` and `supportUrlText` input fields to customize the support button. If not provided no support button will be displayed.
3. Title - customize the page title with the `title` input field. Defaults to `"{organization name} DNS"` if omitted.
4. Colors - customize the primary color (nav bar background color) and the accent color (link and button color) with the `primaryColor` and `accentColor` input fields. Defaults to a bland but functional gray and blueish theme.


**Query**

```graphql
mutation {
  createDnsPortal(
    input: {
      organizationId: "vwvgDqn4kxLk58fsSgGAndh8z"
      name: "example-production"
      title: "Example DNS Portal"
      returnUrl: "https://example.com/customer/123"
      returnUrlText: "Back to Example"
      supportUrl: "https://help.example.com/dns"
      supportUrlText: "Help"
    }
  ) {
    dnsPortal {
      id
      name
    }
  }
}

```

**Response**

```json
{
  "data": {
    "createDnsPortal": {
      "dnsPortal": {
        "id": "P4XYgBwwqqKHwKVmGeaznAgGu6D28nV",
        "name": "example-production"
      }
    }
  }
}
```


## Create a DNS Portal Session

The [`createDnsPortalSession`](https://api.fly.io/graphql/docs/mutation/creatednsportalsession/) mutation creates a time-limited session for the given dns portal and domain. The input requires a [`DNSPortal` node id](https://api.fly.io/graphql/docs/object/organization/#id) and a [`Domain` node id](https://api.fly.io/graphql/docs/object/domain/#id). The response object contains the portal session URL.

You can optionally customize individual sessions with the [`title`](https://api.fly.io/graphql/docs/input_object/creatednsportalsessioninput/#title), [`returnUrl`](https://api.fly.io/graphql/docs/input_object/creatednsportalsessioninput/#returnurl), and [`returnUrlText`](https://api.fly.io/graphql/docs/input_object/creatednsportalsessioninput/#returnurltext) input fields. If not provided the portal's default values will be used.

**Query**

```graphql
mutation {
  createDnsPortalSession(
    input: {
      dnsPortalId: "60RjyPqOlGVVVuDjAdsJp5b"
      domainId: "VJMgmOfdDA45N4kT809"
    }
  ) {
    dnsPortalSession {
      id
      url
    }
  }
}
```

**Response**

```json
{
  "data": {
    "createDnsPortalSession": {
      "dnsPortalSession": {
        "id": "G19dsfdfAZPjGVb42aHVOLmO4bOwlZ1iNYkggM",
        "url": "https://portal.fly.io/dns/44227101de820cebae419f804838753e6b94567bcf6f951272b0d5f9ac31735d"
      }
    }
  }
}
```

## Lookup a portal and domain

Creating a DNS Portal Session requires a DNSPortal node id and a Domain node id. You can look both of those up as needed using the organization slug, portal name, and the domain name.

**Query**

```graphql
query {
  organization(slug: "example-org") {
    dnsPortal(name: "example-production") {
      id
    }
  }
  domain(name: "example.com") {
    id
  }
}
```

**Response**

```json
{
  "data": {
    "organization": {
      "dnsPortal": {
        "id": "60RjyPqOdfgdlGVVVuDjAJp5b"
      },
      "domain": {
        "id": "jwjGDMyzsdfsdex2eOhYV0"
      }
    }
  }
}
```


