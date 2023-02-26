---
layout: default
title: Key Concepts
nav_order: 2
---

# Key Concepts

Before digging into details of the `react-firebase-state` library, it is helpful
to understand the conceptual model illustrated below.

![Conceptual Model](../assets/images/conceptual_model.png)

## Entity
We use the term `Entity` for any data element, whether server-side or client-side.
React components use entities to render views of the data.

## Cache
Entities are stored in a local `Cache`.

## Lease
Certain entities, such as data objects from Firestore documents, are governed by a `Lease`.
The Lease maintains a ledger of all the components that require the Entity for rendering. 
When a component is added to the ledger, we say that the component *claims* the associated
Entity.  When a component is removed from the ledger we say that its claim has been *released*.

Components claim entities by invoking `useEntity`, `watchEntity` or `setLeasedEntity`.

The `useReleaseAllClaims` hook releases all claims held by a given component when that component
unmounts.

It is also possible to release a specific claim by invoking `releaseClaim`.

A leased entity remains in the cache as long as there exists at least one component that claims
the entity.  When there are no more claims, the entity is not evicted from the cache immediately.
Instead, the entity will remain in the cache for a certain amount of time until it is 
garbage collected. The amount of time that an unclaimed entity remains in the cache is given by 
the `abandonTime` configuration parameter which is set to five minutes by default.

The policy of keeping unclaimed entities in the cache for a certain amount of time means that the
user may navigate away from a component briefly and then come back and the required entities will
still be available immediately without having to fetch them again from the server.

When the user navigates back to the component, a new claim is established, and consequently the leased 
entity is no longer eligible for garbage collecting.

## Component

The discussion above assumes that components are React components, and that is usually true but 
not strictly required. Leases reference components by name.  You can give a name to any logical part 
of the application and arrange for that part to claim entities.

A component that has a claim on some entity is called a *leasee*.

## EntityApi
Nearly all of the capabilities of the `react-firebase-state` library are exposed through hooks or
functions that leverage an `EntityApi` instance to do their work.

The `EntityApi` provides access to the Cache, Leases and the FirebaseApp object.

## LeaseeApi
The `LeaseeApi` extends the `EntityApi` interface by providing a reference to a particular
leasee. This is useful when making changes to the cache in callback functions.



