---
layout: default
title: Document Listeners
parent: Guides
nav_order: 2
---

# Document Listeners
{: .no_toc }

<details open markdown="block">
  <summary>
    Table of contents
  </summary>
  {: .text-delta }
- TOC
{:toc}
</details>

## Overview

If you haven't read the [Get Started] article, we recommend that you do so before continuing
with this guide.

The [Get Started] article introduced the following snippet:

```jsx
    const [city, cityError, cityStatus] = useDocListener("SomeComponent", ["cities", cityId]);
```

This guide expands on that example by
- Explaining how to use a type-safe version of `useDocListener`.
- Describing the optional parameters that `useDocListener` accepts.
- Providing guidelines for using the `useDocListener` hook effectively.
- Showing how to start a document listener from an event handler.
- Showing how to release claims on individual entities.

## Type-safe document listener

If you are using typescript, you can add a template parameter to the `useDocListener` hook
as shown below.
```tsx
    const [city, cityError, cityStatus] = useDocListener<City>("SomeComponent", ["cities", cityId]);
```
In this case, the `city` variable will be supplied as an object of type `City`.

## Optional Parameters of `useDocListener`

The `useDocListener` hook accepts a third argument containing optional parameters, so you can 
invoke it like this:
```javascript
    const [city, cityError, cityStatus] = useDocListener(
        "SomeComponent", 
        ["cities", cityId],
        {
            transform: cityTransform,
            onRemoved: handleRemoved,
            onError: handleError,
            leaseOptions: {
                abandonTime: 60000
            }
        }
    );
```
This example introduces four new parameters:
- [transform]
- [onRemoved]
- [onError]
- [leaseOptions]

All of these parameters are optional. We discuss them in detail below.

### transform

The transform parameter allows you to change the raw data received from a
Firestore document into a different data shape for use in your application.

For example, suppose a city document from Firestore looks like this:
```javascript
{
    cityName: "Liverpool",
    councillors: {
        UOcUt: {id: "UOcUt", name: "Pat Moloney",     ward: "Childwall"  },
        PxFNV: {id: "PxFNV", name: "Ellie Byrne",     ward: "Everton"    },
        vQwxK: {id: "vQwxK", name: "Lynnie Hinnigan", ward: "Cressington"}
    }
}
```
In this document, councillors are stored in a map indexed by the
councillor's id.  Let's suppose that a document having this structure
matches the `ServerCity` type.

On the client, it might be more convenient to have the councillors in an array
sorted by name as shown below:
```javascript
{
    id: "dIjZC",
    cityName: "Liverpool",
    councillors: [
        {id: "PxFNV", name: "Ellie Byrne",     ward: "Everton"  },
        {id: "vQwxK", name: "Lynnie Hinnigan", ward: "Cressington"},
        {id: "UOcUt", name: "Pat Moloney"      ward: "Childwall"}
    ]
}
```
A document having this structure matches the `ClientCity` type.

To convert a `ServerCity` into a `ClientCity`, we introduce the following transform.
```typescript
function cityTransform(event: DocChangeEvent<ServerCity>): ClientCity {
    const serverData = event.data;
    const path = event.path;
    const councillors  = Object.values(serverData.councillors);
    councillors.sort( (a, b) => a.name.localeCompare(b.name) );

    return {
        id: path[path.length-1],
        cityName: serverData.cityName,
        councillors
    }
}
```

The transform function is used as shown below.

```javascript
    const [city, cityError, cityStatus] = useDocListener(
        "SomeComponent", ["cities", cityId], {transform: cityTransform}
    );
```

In this case, the `city` variable will be returned as an object of type `ClientCity`.

When a `transform` function is given to the `useDocListener` hook, you don't need to 
supply a template parameter to specify the entity type. The typescript compiler will
figure it out from the signature of your transform function.

The transform function is called when the document is initially received from
Firestore and whenever the document is modified. The return value from the transform
function is stored in the cache (not the raw data from the Firestore document).

### onRemoved

The `onRemoved` parameter allows you to define a callback function that fires whenever
the Firestore document is deleted.

Here's a snippet showing how to use this parameter.
```typescript
    function handleRemoved(event: DocRemovedEvent<ServerCity>) {
        const serverData = event.data;
        const message = `The city "${serverData.cityName}" has been deleted`;
        console.log(message);
    }
    const [city, cityError, cityStatus] = useDocListener<City>(
        "SomeComponent", ["cities", cityId], {onRemoved: handleRemoved}
    );
```
The `handleRemoved` function defined here merely logs a message to the console. In a production 
app, you might consider rendering an "info" message that notifies the user that the city entity 
was deleted.

See [Manage client-side state] for an example that shows how you can implement an alerting
system.

### onError
The `onError` parameter allows you to define a callback function that fires if an error
occurred while fetching the document from Firestore.

Here's a snippet showing how to use this parameter.
```typescript
    function handleError(event: DocErrorEvent) {
        const path = event.path;
        const cityId = path[path.length-1];
        const message = `An error occurred while loading the city[id=${cityId}]`;
        console.error({message, event.error});
    }
    const [city, cityError, cityStatus] = useDocListener<City>(
        "SomeComponent", ["cities", cityId], {onError: handleError}
    );
```
The `onError` function defined here merely logs to the console. In a production
app, you really should render an error message that the user will notice.

Again, see [Manage client-side state] for  details about implementing an alerting
system.

### leaseOptions

The `leaseOptions` parameter is an object that customizes the behavior of the lease created
for the entity. Currently, there is only one possible field within the `leaseOptions` namely 
`abandonTime`. This value specifies the number of milliseconds that the entity will remain in 
the cache after all components have released their claims on that entity.

Here's a snippet showing how to use the `leaseOptions` parameter.
```typescript
   
    const [city, cityError, cityStatus] = useDocListener(
        "SomeComponent", ["cities", cityId], {leaseOptions: {abandonTime: 60000}}
    );
```

If you want the entity to remain in the cache indefinitely, set `abandonTime` equal to
`Number.POSITIVE_INFINITY`.

## Guidelines for using the `useDocListener` hook

There are several things you should know about the `useDocListener` hook.
- The document listener is started the first time the hook is called for a given path.
- The optional parameters are only used when the document listener is started. This means
  that the optional parameters are ignored after the first call to the hook for a given path.
- All components that invoke `useDocListener` will establish a claim on the specified entity.
- Entities become eligible for garbage collection once all claims have been released.
- If the document specified by the `path` does not exist or cannot be read due to Firestore
  secuity rules, then `useDocListener` will return an error whose message contains the string
  "Missing or insufficient permissions".

It is important to follow the guidelines listed below:
- If multiple clients invoke `useDocListener`, make sure they all use the same `transform`.
    This is important because the cache will store only one instance of a given entity.
    If different components use different transforms you will end up with type conflicts.
- Similar comments apply to the other optional parameters (`onRemoved`, `onError` and `leaseOptions`).
    Make sure that all invocations of `useDocListener` for a given path use the same handlers for these
    parameters.

## Using a document listener within event handlers

The `watchEntity` function gives you the same functionality as the `useDocListener` hook, 
but it is designed for use within event handlers. There are two types of event handlers to 
consider.

First, we have Firestore event handlers. These are handlers
that fire when a Firestore document changes state, and they are specified as the optional
parameters of the `useDocListener` hook:
- [transform] fires when the document is first loaded or is modified
- [onRemoved] fires when the document is removed from Firestore
- [onError] fires if an error occurs while fetching the document from Firestore

Second, we have HTML event handlers such as the `onClick` handler for a button.

We illustrate the usage patterns for both types of event handlers below.

### Using `watchEntity` within a Firestore event handler

Suppose that city documents in Firestore match the type `ServerCity`, and they 
look like this:
```javascript
// Document: cities/dIjZC
{
    cityName: "Liverpool",
    councillors: {
        UOcUt: true,
        PxFNV: true,
        vQwxK: true
    }
}
```
In this scenario, the set of city councillors is represented as a map
where the key is the `id` of the councillor, and the value is the 
boolean literal `true`. Information about individual councillors can be found
in the `councillors` collection which stores documents of type 
`ServerCouncillor`, defined as follows:
```typescript
interface ServerCouncillor {
    cityId: string;
    name: string;
    ward: string;
}
```
Here's an example of a councillor document stored in Firestore:
```javascript
// Document: councillors/PxFNV
{
    cityId: "dIjZC",
    name: "Ellie Byrne", 
    ward: "Everton"
}
```

We require a `ClientCouncillor` type that extends `ServerCouncillor` by
adding the councillor `id`. Thus, we have:
```typescript
interface ClientCouncillor extends ServerConcillor {
    id: string
}
```

Here's an example of the city data that will be stored in the local cache:
```javascript
{
    id: "dIjZC",
    cityName: "Liverpool",
    councillors: [
        {id: "PxFNV", name: "Ellie Byrne",     ward: "Everton"    , cityId: "dIjZC"},
        {id: "vQwxK", name: "Lynnie Hinnigan", ward: "Cressington", cityId: "dIJZC"},
        {id: "UOcUt", name: "Pat Moloney"      ward: "Childwall"  , cityId: "dIJZC"}
    ]
}
```

We can use the following code to satisfy the requirements of the current scenario.

```typescript
/**
 * Sort an array of concillors by name
 */
function sortCouncillors(councillors: ClientCouncillor[]) {
    councillors.sort( (a, b) => a.name.localeCompare(b.name));
}

function councillorTransform(event: DocChangeEvent<ServerCouncillor>) {
    const serverData = event.data;
    const path = event.path;
    const id = path[path.length-1];
    const councillor: ClientCouncillor = {
        ...serverData,
        id
    }

    const cityPath = ["cities", councillor.cityId];
    const [city] = getEntity<ClientCity>(api, cityPath);

    if (city) {
        const councillors = city.councillors.filter( c => c !== id);
        councillors.push(councillor);
        sortCouncillors(councillors);
        setEntity(api, cityPath, {...city, councillors});
    }
    return councillor;
}

function cityTransform(event: DocChangeEvent<ServerCity>) : ClientCity {
    const api = event.api; // The EntityApi instance used by the application
    const serverData = event.data;
    const leasee = event.leasee;
    const path = event.path;
    const councillors: ClientCouncillor[] = [];

    for (const councillorId in serverData.councillors) {
        const councillorPath = ["councillors", councillorId];
        const [councillor, error] = watchEntity(
            api, leasee, councillorPath, {transform: councillorTransform}
        )
        if (error) {
            console.error(`Failed to load councillor[id=${councillorId}]`, error);
        }
        if (councillor) {
            councillors.push(councillor);
        }
    }
    sortCouncillors(councillors);

    return {
        id: path[path.length-1],
        name: serverData.name,
        councillors
    }
}
```

The `watchEntity` function accepts the same optional parameters as `useDocListener`.

### Using `watchEntity` within an HTML event handler

In this example, a component renders the name of a given city
when a button is pressed. Yes, it's a somewhat contrived example.

```tsx
import { 
    useEntityApi, useEntity, watchEntity, useReleaseAllClaims 
} from "@gmcfall/react-firebase-state";

/**
 * A React Component that renders the city name when a button is pressed
 */
function CityName({cityId}) {
    const path = ["cities", cityId];
    const api = useEntityApi();
    const [city,,cityStatus] = useEntity<City>(path);

    useReleaseAllClaims("CityName");

    function handleClick() {
        watchEntity(api, "CityName", path);
    }

    const text = (
        cityStatus==="pending" ? "Loading..." :
        cityStatus==="error"   ? "Oops! An error occurred." :
        cityStatus==="deleted" ? "Oops! The city was deleted from the database" :
        city                   ? city.name :
        ""
    )

    return (
        <>
            <button onClick={handleClick}>
                Show the city name
            </button>
            <span>{text}</span>
        </>
    )
}
```
----
[Get Started]: ../..
[transform]: #transform
[onRemoved]: #onRemoved
[onError]: #onerror
[leaseOptions]: #leaseoptions
[Manage client-side state]: ./manage_client_side_state.html