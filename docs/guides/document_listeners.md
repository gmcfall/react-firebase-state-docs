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
    const [cityStatus, city, cityError] = useDocListener("SomeComponent", ["cities", cityId]);
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
    const [cityStatus, city, cityError] = useDocListener<City>("SomeComponent", ["cities", cityId]);
```
In this case, the `city` variable will be supplied as an object of type `City`.

## Optional Parameters of `useDocListener`

The `useDocListener` hook accepts a third argument containing optional parameters, so you can 
invoke it like this:
```javascript
    const [cityStatus, city, cityError] = useDocListener(
        "SomeComponent", 
        ["cities", cityId],
        {
            transform: cityTransform,
            onRemove: handleRemove,
            onError: handleError,
            leaseOptions: {
                cacheTime: 60000
            }
        }
    );
```
This example introduces four new parameters:
- [transform]
- [onRemove]
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
function cityTransform(api: LeaseeApi, serverData: ServerCity, path: string[]): ClientCity {
    const councillors = Object.values(serverData.councillors);
    councillors.sort( (a, b) => a.name.localeCompare(b.name) );

    return {
        id: path[path.length-1],
        cityName: serverData.cityName,
        councillors
    }
}
```

Transform functions take three arguments:
- `api`: A LeaseeApi
- `serverData`: The raw data from the Firestore document
- `path`: The path to the Firestore document

The `api` parameter allows the transform function to read other entities in the cache or
modify them as a side effect.

The `cityTransform` function does not use the `api` parameter, but it still must
declare it.

The transform function is used as shown below.

```javascript
    const [cityStatus, city, cityError] = useDocListener(
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

### onRemove

The `onRemove` parameter allows you to define a callback function that fires whenever
the Firestore document is deleted.

Here's a snippet showing how to use this parameter.
```typescript
    function handleRemove(api: LeaseeApi, serverData: ServerCity, path: string[]) {
        const message = `The city "${serverData.cityName}" has been deleted`;
        console.log(message);
    }
    const [cityStatus, city, cityError] = useDocListener<City>(
        "SomeComponent", ["cities", cityId], {onRemove: handleRemove}
    );
```
The `handleRemove` function defined here merely logs a message to the console. In a production 
app, you might consider rendering an "info" message that notifies the user that the city entity 
was deleted.

See [Manage client-side state] for an example that shows how you can implement an alerting
system.

### onError
The `onError` parameter allows you to define a callback function that fires if an error
occurred while fetching the document from Firestore.

Here's a snippet showing how to use this parameter.
```typescript
    function handleError(api: LeaseeApi, error: Error, path: string[]) {
        const cityId = path[path.length-1];
        const message = `An error occurred while loading the city[id=${cityId}]`;
        console.error({message, error});
    }
    const [cityStatus, city, cityError] = useDocListener<City>(
        "SomeComponent", ["cities", cityId], {onError: handleError}
    );
```
The `onError` function defined here merely logs to the console. In a production
app, you really should render an error message that the user will notice.

Again, see [Manage client-side state] for  details about implementing an alerting
system.

### leaseOptions

The `leaseOptions` parameter is an object that customizes the behavior of the lease created
for the entity. Currently, there is only possible field within the `leaseOptions` namely `cacheTime`.
This value specifies the number of milliseconds that the entity will remain in the cache after
all components have released their claims on that entity.

Here's a snippet showing how to use the `leaseOptions` parameter.
```typescript
   
    const [cityStatus, city, cityError] = useDocListener(
        "SomeComponent", ["cities", cityId], {leaseOptions: {cacheTime: 60000}}
    );
```

If you want the entity to remain in the cache indefinitely, set `cacheTime` equal to
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
- Similar comments apply to the other optional parameters (`onRemove`, `onError` and `leaseOptions`).
    Make sure that all invocations of `useDocListener` for a given path use the same handlers for these
    parameters.
- Don't forget to call `releaseEntities` when your component unmounts.

## Using a document listener within event handlers

The `watchEntity` function gives you the same functionality as the `useDocListener` hook, 
but it is designed for use within event handlers. There are two types of event handlers to 
consider.

First, we have Firestore event handlers. These are handlers
that fire when a Firestore document changes state, and they are specified as the optional
parameters of the `useDocListener` hook:
- [transform] fires when the document is first loaded or is modified
- [onRemove] fires when the document is removed from Firestore
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
`ServerCouncillor` 
with data of the form:

```javascript
// Document: councillors/PxFNV
{
    cityId: "dIjZC",
    name: "Ellie Byrne", 
    ward: "Everton"
}
```

We require that the `ClientCity` type has the form:

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

We have introduced the `ClientCouncillor` interface which extends `ServerCouncillor`
by adding the councillor's `id` value.

We can use the following code to satisfy the requirements of the current scenario.

```typescript
function sortCouncillors(councillors: string[]) {
    councillors.sort( (a, b) => a.name.localeCompare(b.name));
}

function councillorTransform(api: LeaseeApi, serverData: ServerCouncillor, path: string[]) {

    const id = path[path.length-1];
    const councillor: ClientCouncillor = {
        ...serverData,
        id
    }

    const cityPath = ["cities", councillor.cityId];
    const [,city] = getEntity<ClientCity>(api, cityPath);

    if (city) {
        const councillors = city.councillors.filter( c => c !== id);
        councillors.push(councillor);
        sortCouncillors(councillors);
        setEntity(api, cityPath, {...city, councillors});
    }
    return councillor;
}

function cityTransform(api: LeaseeApi, serverData: ServerCouncillor, path: string[]) : ClientCity {
    const client = api.getClient();
    const leasee = api.leasee;
    const councillors: ClientCouncillor[] = [];

    for (const councillorId in serverData.councillors) {
        const councillorPath = ["councillors", councillorId];
        const [, councillor, error] = watchEntity(
            client, leasee, councillorPath, {transform: councillorTransform}
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
import { useEntityApi, useEntity, watchEntity, releaseEntities } from "@gmcfall/react-firebase-state";

function CityName({cityId}) {
    const path = ["cities", cityId];
    const api = useEntityApi();
    const [cityStatus, city] = useEntity<City>(path);

    useEffect(() => () => releaseEntities("CityName"), []);

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
[onRemove]: #onremove
[onError]: #onerror
[leaseOptions]: #leaseoptions
[Manage client-side state]: ./manage_client_side_state.html