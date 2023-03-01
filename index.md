---
title: Get Started
layout: home
nav_order: 1
---
# Get Started

`react-firebase-state` is a component library that helps you manage server-side state from *Firebase Auth* and *Firestore* 
within a React web app. It also provides utilities for managing client-side state that you control.

## Installation

To use `react-firebase-state`, you must install four peer dependencies if you don't already have them:
- `react`
- `react-dom`
- `immer`
- `firebase`

In your react project folder, use npm to install these dependencies:
```sh
npm install react react-dom immer firebase
```

Then install the `react-firebase-state` component library.
```sh
npm install @gmcfall/react-firebase-state
```
## Prerequisites

This documentation assumes that you are familiar with [React] and [Firebase].
Most of the examples assume some knowledge of Typescript, but you can use the 
library with plain-old Javascript.

## Using the library

The `react-firebase-state` library provides a collection of hooks and other utilities
for managing state. To use the library in the child components of your application, 
you must wrap them within a `<FirebaseProvider>` component.

### Wrap your application

```tsx
import { FirebaseProvider } from '@gmcfall/react-firebase-state';

// You must initialize the FirebaseApp instance that your application uses
// and pass it to the <FirebaseProvider> component.

// It is not necessary to implement a function called "initializeFirebaseApp".  
// You could initialize Firebase in a module and simply export `firebaseApp` 
// as a constant. How you perform the initialization is up to you.

const firebaseApp = initializeFirebaseApp();

export function App() {

    return (
        <FirebaseProvider firebaseApp={firebaseApp}>
           { /* Add your child components here*/ }
        </FirebaseProvider>
    )
}

```

### Usage in child components

There are lots of ways to use the `react-firebase-state` library. Here, we merely 
illustrate two common use cases:

- [Access the current user](#access-the-current-user)
- [Listen for changes to a document](#listen-for-changes-to-a-firestore-document)

For other use cases, explore the list of available guides.


#### Access the current user

This example shows how you can get information about the currently authenticated user.

```jsx
import { useAuthListener } from '@gmcfall/react-firebase-state';

export function ComponentThatAccessesTheCurrentUser() {
    
    const [user, userError, userStatus] = useAuthListener();

    switch (userStatus) {
        case "pending":
            // Information about the current user is being fetched
            // asynchronously. You might want to render a spinner.
            // `user` and `userError` are undefined. 
            break;

        case "signedIn":
            // The current user is signed in. 
            // The `user` variable contains the Firebase `User` object.
            // `userError` is undefined.
            break;        

        case "signedOut":
            // The current user is signed in. 
            // The `user` variable holds the  Firebase `User` object.
            // `userError` is undefined
            break;

        case "error":
            // An error occurred while fetching information about the user.
            // `user` is undefined.
            // `userError` contains the Error thrown by Firebase
            break;
    }

    // ...
}
```

#### Listen for changes to a Firestore document

In this example, a component listens for changes to a Firestore document
containing information about a city. The id for the city document is
passed via props.

```jsx
import { useEffect } from "react";
import { useDocListener } from '@gmcfall/react-firebase-state';

export function SomeComponent({ cityId }) {

    const [city, cityError, cityStatus] = useDocListener("SomeComponent", ["cities", cityId]);

    switch (cityStatus) {

        case "idle":
            // The path passed to `useDocListener` contains an undefined
            // value, and therefore a document listener was not started
            // `city` and `cityError` are both undefined.
            break;

        case "pending":
            // The document is being fetched asynchronously and
            // the response is pending.
            // The `city` and `cityError` variables are undefined.
            break;

        case "success":
            // The document was successfully retrieved from Firestore.
            // The `city` variable contains the document data.
            // `cityError` is undefined.
            break;

        case "removed":
            // The document was removed from Firestore.
            // The `city` variable is null.
            // `cityError` is undefined.
            break;

        case "error":
            // An error occurred while fetching the document from Firestore.
            // `city` is undefined.
            // `cityError` contains the Error thrown by Firestore.
            break;
    }
}
```
This is just a rough skeleton. A real implementation would return an 
appropriate child component for each case in the switch statement. 

The `useDocListener` hook does two things:

1. It starts a listener for the specified Firestore document and returns the 
   document data (or an error).
2. It establishes a lease on the document data.

As long as there is at least one component holding a lease, the document data will 
remain in a local cache and be updated when changes occur.

The first argument to `useDocListener` is the name of the lease holder.  This name can be
anything, but it is a best practice to use the name of the component.

The second argument is the path to the target document expressed as an array of strings. 
In this example, "cities" is the name of the Firestore collection and `cityId` is the 
document id, so `["cities", cityId]` is the path to the document.

When the component unmounts, the `useEffect` hook releases all of the component's leases.

## Next Steps
- Read about [Key Concepts]
- Explore the [Guides]

----
[React]: https://reactjs.org/
[Firebase]: https://firebase.google.com/
[Key Concepts]: ./docs/key_concepts.html
[Guides]: ./guides.html