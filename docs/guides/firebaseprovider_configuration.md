---
layout: default
title: FirebaseProvider Configuration
parent: Guides
nav_order: 1
---

# FirebaseProvider Configuration

In the [Get Started] article, you learned how to wrap your app with the
`FirebaseProvider` component using the default configuration.

You can override the defaults as shown below.

```tsx
import { FirebaseProvider } from '@gmcfall/react-firebase-state';

// You must initialize the FirebaseApp instance that your application uses
// and pass it to the <FirebaseProvider> component.
//
// It is not necessary to implement a function called "initializeFirebaseApp".  
// You could initialize Firebase in a module and simply export `firebaseApp` 
// as a constant. How you perform the initialization is up to you.

const firebaseApp = initializeFirebaseApp();

// By default, the FirebaseProvider uses an empty object as the initial
// state for client-side entities.  You may optionally define your own initial
// state populated with default entities.
//
// It is not necessary to implement a function called "createInitialState".
// How you create the initial state object is up to you.

const initialState = createInitialState();

// The `abandonTime` parameter specifies the number of milliseconds that a leased
// entity can linger in the cache without any leasees before it becomes eligible
// for garbage collection. The default value of `abandonTime` is `300000` (5 minutes).
// you can override this value by passing `options` to the FirebaseProvider.

const options = {
    abandonTime: 120000 // 2 minutes
}

export function App() {

    return (
        <FirebaseProvider 
            firebaseApp={firebaseApp}
            initialState={initialState}
            options={options}
        >
           { /* Add your child components here*/ }
        </FirebaseProvider>
    )
}

```
The `firebaseApp` property is required, but the `initialState` and `options` properties are optional.

----
[Get Started]: ../..