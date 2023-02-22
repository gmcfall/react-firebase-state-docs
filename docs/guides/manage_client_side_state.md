---
layout: default
title: Manage Client-Side State
parent: Guides
nav_order: 3
---

# Manage Client-Side State

React applications can manage client-side state by taking advantage
of the [useState] hook (and potentially drilling state down into child 
components) or leveraging the [React Context] feature.

The `react-firebase-state` library offers an alternative to these "standard"
client-side state management solutions.

This guide illustrates `react-firebase-state` client-side state 
management by showing how it can be used to display error messages,
warnings, info messages, and success messages.

There are three parts to the client-side state management solution:

- [Defining an interface for your client-side state]
- [Implementing React components that use client-side state]
- [Mutating client-side state]

We discuss each part of the solution below.

## Define an interface for client-side state

We highly recommend using Typescript for your application, but if you are
using plain-old Javascript, you can skip this step.

If you are using Typescript, you'll want to define an interface (or `type`)
for your client-side data.

Our alerting solution provides the following definitions.
```tsx
// File: ./src/types.ts

export type AlertSeverity: 'error' | 'warning' | 'info' | 'success';

export interface AlertData {
    severity: AlertSeverity;
    message: string
}

export interface SampleApp {
    alertData?: AlertData;
}
```
The `SampleApp` interface defines the structure of the client-side data 
used in this guide.  It declares a single entity of type `AlertData`.
A more complex app might declare other client-side entities.

## Implementing React components that use client-side state

Here we present the code for an `<Alert>` component that renders `AlertData`.
If you are using [Material Design], you might include this component in your [App Bar].

The `<Alert>` component looks for the `alertData` entity. If it exists, the component
renders the encapsulated message with an animation that fades in and then fades out 
after 10 seconds.

The code is spread across three files.

The first file defines a CSS class used by the component. This class implements
the animation.
```css
/* File: ./src/components/Alert/alert.css */

.alert {
  animation: alertTransition 10s ease-in forwards;
}

@keyframes alertTransition {
  0% {
    opacity: 0;
  }

  10% {
    opacity: 1;
  }

  90% {
    opacity: 1
  }

  100% {
    opacity: 0
  }  
}
```

The second file defines functions for manipulating the component.
```ts
// File: ./src/components/Alert/alertApi.ts

import { EntityApi } from "@gmcfall/react-firebase-state";
import { SampleApp } from "../../types";

/**
 * A selector for accessing `alertData` from the `SampleApp`
 */ 
export function selectAlert(app: SampleApp) {
    return app.alertData;
}

/**
 * Delete the `alertData` entity from the client state
 */
export function alertRemove(api: EntityApi) {
    api.mutate(
        (app: SampleApp) => {
            delete app.alertData
        }
    )
}

/**
 * A helper function for setting an error message in the `alertData` entity.
 * This function allows the caller to set the error message by passing an 
 * and `EntityApi` instance. Mutation of client state is handled internally
 * as a side-effect of the function.
 * 
 * Example:
 * ```
 *   handleError(api: LeaseeApi, error: Error, path: string[]) {
 *      alertError(api, "Oops! An error occurred", error);
 *   }
 * ```
 * 
 * A caller that is explicitly mutating the client state should use `setError`
 * instead.
 */
export function alertError(api: EntityApi, message: string, error?: unknown, context?: any) {
    api.mutate((app: SampleApp) => setError(app, message, error, context))
}

/**
 * A helper function for setting an error message in the `alertData` entity.
 * This function should be called from within an `EntityApi.mutate` callback.
 */
export function setError(app: SampleApp, message: string, error?: unknown, context?: any) {

    const extraInfo = (
        (context && error) ? {...context, error} :
        context ? {...context} :
        error? {error} :
        null
    )
    if (extraInfo) {
        console.error(message, extraInfo);
    } else {
        console.error(message);
    }

    app.alertData = {severity: "error", message}
}

// The remainder of this file contains functions for setting
// "warning", "info", and "success" messages.
//
// These functions follow the same pattern that was used for
// "error" messages.

// -------------------------------------------------------------
export function alertWarning(api: EntityApi, message: string) {
    api.mutate((app: SampleApp) => setWarning(app, message));
}

export function setWarning(app: SampleApp, message: string) {
    app.alertData = {severity: "warning", message}
}
// -------------------------------------------------------------
export function alertInfo(api: EntityApi, message: string) {
    api.mutate((app: SampleApp) => setInfo(app, message));
}

export function setInfo(app: SampleApp, message: string) {
    app.alertData = {severity: "info", message}
}
// -------------------------------------------------------------
export function alertSuccess(api: EntityApi, message: string) {
    api.mutate(
        (app: SampleApp) => setSuccess(app, message)
    )
}

export function setSuccess(app: SampleApp, message: string) {
    app.alertData = {severity: "success", message}
}

```
Some functions in the `alertApi.ts` file invoke the `mutate` method
of `EntityApi`. We discuss this method in the [Mutating client-side state] section 
below.

The third file supplies the React `<Alert>` component.
```tsx
// File: ./src/components/Alert/Alert.tsx

import { Alert as MuiAlert} from "@mui/material";
import { useEffect } from 'react';
import { useData, useEntityApi } from "@gmcfall/react-firebase-state";
import { alertRemove, selectAlert } from "./alertApi";
import "./alert.css"

let timeoutId: ReturnType<typeof setTimeout> | null = null;

export default function Alert() {

    const api = useEntityApi();
    const alertData = useData(selectAlert);

    // The following effect removes the current alert message
    // after 10 seconds.

    useEffect(() => {
        // Only one alert message may be displayed at a time.
        // If a new message arrives before the previous message is removed,
        // then `timeoutId` will have a non-null value. In this case,
        // we need to clear the timer that is waiting to remove the
        // old (and now obsolete) message.

        if (timeoutId) {
            clearTimeout(timeoutId);
            timeoutId = null;
        }

        if (alertData) {
            // There was a change to `alertData`.
            // Let's set a timer that will remove the current message
            // after 10 seconds.

            timeoutId = setTimeout(() => {
                alertRemove(api);
                timeoutId = null;
            }, 10000)
        }

    }, [alertData, api])


    if (!alertData) {
        return null;
    }

    return (
        <MuiAlert className="alert" severity={alertData.severity}>
            {alertData.message}
        </MuiAlert>
    );
}
```
Notice that this component leverages the `useData` hook. This hook takes a single argument
which is a function that returns some data from the client-side state. In this example,
the function is `selectAlert` from the `alertApi.ts` file.

As a reminder, here's the definition of that function:
```typescript
export function selectAlert(app: SampleApp) {
    return app.alertData;
}
```
The typescript compiler is smart enough to recognize that the return value is an object
of type `AlertData`.

Consequently, the line
```tsx
    const alertData = useData(selectAlert);
```
yields a value for `alertData` that has the `AlertData` type.

## Mutating client-side state

The [Document Listeners] guide presented the following error handler:

```typescript
    function handleError(api: LeaseeApi, error: Error, path: string[]) {
        const cityId = path[path.length-1];
        const message = `An error occurred while loading the city[id=${cityId}]`;
        console.error({message, error});
    }
```
This handler has the drawback that it merely logs to the console.  It would be
better to display the error message to the user via our alerting system as shown
below.
```typescript
    function handleError(api: LeaseeApi, error: Error, path: string[]) {
        api.mutate(
            (app: SampleApp) => {
                app.alertData = {
                    severity: "error",
                    message: "An error occurred while loading the city data"
                }
            }
        )
    }
```
This revision of the handler uses the `mutate` function provided by the `LeaseeApi`.
The `mutate` function takes a callback as its sole argument. The callback receives
the current client-side state, and the body of the callback makes changes
to that state.

As a best practice, we recommend putting mutations into helper functions. For instance,
we can simplify the error handler by leveraging a helper function from the `alertApi.ts`
file. 
```typescript
    function handleError(api: LeaseeApi, error: Error, path: string[]) {
        const context = {cityId: path[path.length-1]};
        alertError(api, "An error occurred while loading the city data", error, context);
    }
```

Similarly, we can define a Firestore remove handler like this:
```typescript
    function handleRemove(api: LeaseeApi, serverData: ServerCity, path: string[]) {
        const message = `The city "${serverData.cityName}" has been deleted`;
        alertSuccess(api, message);
    }
```

In addition, we can mutate client-side state from within HTML event handlers as shown below.
```tsx
    function MutatorButton() {
        const api = useEntityApi();

        const handleClick() {
            api.mutate(
                (app: SampleApp) => {
                    // Perform your mutations here
                }
            )
        }

        return (
            <button onClick={handleClick}>
                Click Me!
            </button>
        )
    }
```

----
[React Context]: https://reactjs.org/docs/context.html
[useState]: https://reactjs.org/docs/hooks-state.html
[Material Design]: https://m2.material.io/
[App Bar]: https://m2.material.io/components/app-bars-top
[Document Listeners]: ./document_listeners.html
[Defining an interface for your client-side state]: #define-an-interface-for-client-side-state
[Implementing React components that use client-side state]: #implementing-react-components-that-use-client-side-state
[Mutating client-side state]: #mutating-client-side-state