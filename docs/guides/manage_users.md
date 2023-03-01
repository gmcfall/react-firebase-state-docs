---
layout: default
title: Manage Users
parent: Guides
nav_order: 4
---

# Manage Users

{: .prerequisites}
> This guide leverages the alerting mechanism presented in the [Manage client-side state] article.
>
> It also assumes you are familiar with:
> - `useDocListener`
> - `useEntity`
> - `watchEntity`
>
> See [Document Listeners] for more information about these elements of the `react-firebase-state` 
> library.

The `react-firebase-state` library provides tools that make it easy to
- [Listen for changes to the state of the current user]
- [Customize the properties of the user]
- [Get information about the current user]
- [Get information about any user]

We discuss these capabilities below.



## Listen for changes to the state of the current user

If your application needs information about the current user, you'll want to
define a component that triggers the Firebase Auth listener.

We recommend that you implement a component like `FirebaseAuthListener` shown below.
```tsx
// File: ./src/components/FirebaseAuthListener.tsx

import { useAuthLisener, AuthErrorEvent } from "@gmcfall/react-firebase-state";
import { alertError, alertSuccess } from "./Alert/alertApi";

function handleError(event: AuthErrorEvent) {
    const error = event.error;
    alertError(api, "An error occurred while getting information about your account", error);
}

interface FirebaseAuthListenerProps {
    children?: React.ReactNode;
}

export default function FirebaseAuthListener(props: FirebaseAuthListenerProps) {
    const {children} = props;
    useAuthListener({onError: handleError});

    return (
        <>
            {children}
        </>
    )
}
```

You would use the `FirebaseAuthListener` component as shown below.
```tsx
// File: ./src/components/App.tsx

import { FirebaseProvider } from '@gmcfall/react-firebase-state';
import { FirebaseAuthListener } from "./FirebaseAuthListener";

const firebaseApp = initializeFirebaseApp();

export function App() {

    return (
        <FirebaseProvider firebaseApp={firebaseApp}>
            <FirebaseAuthListener>
                { /* Add your child components here*/ }
            </FirebaseAuthListener>
        </FirebaseProvider>
    )
}
```

You could, of course, put the `FirebaseAuthListener` lower within your component hierarchy.
In general, you want it to wrap any child components that require information about the 
current user.

## Customize the properties of the user

Your application may need to customize the properties of the current user.
In this section, we show how to enrich the user data with a Twitter-like 
handle.

There are four steps involved in this solution.
- [Define an interface for the enriched user]
- [Define a Firebase collection that stores the extra user properties]
- [Define an api for managing the enriched user data]
- [Use the api to transform the Firebase User into the enriched user]

### Define an interface for the enriched user

Let's call the enriched user a `SessionUser`.  Here's the interface definition:
```typescript
// File: ./src/shared/types.ts

import { UserMetadata, UserInfo } from "firebase/auth";

export interface SessionUser {
    // Properties from Firebase User
    emailVerified: boolean;
    isAnonymous: boolean;
    metadata: UserMetadata;
    providerData: UserInfo[];
    refreshToken: string;
    tenantId: string | null;

    /** A Twitter-like handle for the user */
    handle: string;
}
```

### Define a Firebase collection that stores the handles

In our example, we define a Firebase collection named "identities" that stores 
more than just Twitter-like handles for users.  In particular, it stores the
new `handle` property plus the `uid` and `displayName` from the Firebase `User`.

Documents in the "identities" collection conform to the `Identity` interface
shown below.
```typescript
// File: ./src/shared/types.ts

export interface Identity {
    /** The `uid` property for the user as defined by the Firebase Auth system */
    uid: string;

    /** The `displayName`  for the user as defined by the Firebase Auth system. */
    displayName: string;

    /** A custom Twitter-like handle for the user */
    handle: string;
}
```
The app must persist these documents as part of the user registration process.

The `Identity` interface is convenient because the application may need to render the `displayName` 
for users other than the current user, and it is non-trivial to extract that information from the 
Firebase Auth system. (You need to implement a Firebase Function and use the Admin SDK.) 

{: .warning}
> The `displayName` property defined by the Firebase Auth system may contain the user's real
> name, and it may therefore be classified as sensitive information. Similarly, the user 
> may choose a `handle` that contains personally identifiable information. 
>
> If you use the solution presented in this guide in a production system, make sure that you
> comply with the relevant regulations regarding the protection of personal information.

### Define an api for managing the enriched user data

We need to satisfy the following requirements:
1. The `displayName` property in `Identity` documents must remain synchronized
   with the Firestore `User`.
2. Whenever the state of the current `User` changes, we must update the record
   of the user in the local cache so that it matches the `SessionUser` interface.

The following functions support these requirements.

```typescript
// File: ./src/shared/identity.ts

import { getAuth } from "firebase/auth";
import { getFirestore, doc, updateDoc } from "firebase/firestore";
import { EntityApi, setAuthUser, watchEntity } from "@gmcfall/react-firebase-state";
import { alertError } from "../components/Alert/alertApi";
import { Identity, SessionUser } from "./types";

function createSessionUser(user: User, identity: Identity): SessionUser {
    return {
        emailVerified: user.emailVerified,
        isAnonymous: user.isAnonymous,
        metadata: user.metadata,
        providerData: user.providerData,
        refreshToken: user.refreshToken,
        tenantId: user.tenantId,
        handle: identity.handle
    }
}

/**
 * Update the Firestore `Identity` document for a given user so that the 
 * `displayName` is consistent with Firebase Auth.
 */
async function updateDisplayName(api: EntityApi, user: User) {
    const userUid = user.uid;
    const displayName = user.displayName;
    const db = getFirestore(api.firebaseApp);
    const identityRef = doc(db, "identities", userUid);
    try {
        await updateDoc(identityRef, {displayName});
    } catch (error) {
        const message = "An error occurred while updating your user profile";
        alertError(api, message, error, {userUid});
    }
}

/**
 * A transform for Identity entities which updates the current user entity
 * in the local cache as a side-effect.
 * 
 * This function does not actually transform the Identity entity. It sole purpose
 * is to keep the current user entity up-to-date and consistent with the 
 * SessionUser interface.
 */
function transformIdentity(event: DocChangeEvent<Identity>) {
    const api = event.api;
    const serverData = event.data;
    const auth = getAuth(api.firebaseApp);
    const user = auth.currentUser;
    if (user && user.uid === serverData.uid) {
        if (user.displayName !== serverData.displayName) {
            updateDisplayName(api, user);
        }
        const sessionUser = createSessionUser(user, identity);
        setAuthUser(api, sessionUser);
    }

    return serverData
}

function handleIdentityError(event: DocErrorEvent) {
    const api = event.api;
    const error = event.error;
    const path = event.path;
    const auth = getAuth(api.firebaseApp);
    
    const message = auth.currentUser ?
        "An error occurred while loading your user profile" :
        "An error occurred while loading the profile for another user";

    alertError(api, message, error, {path});
}

export const IDENTITY_OPTIONS = {
    transform: transformIdentity,
    onError: handleIdentityError
}

export function identityPath(userUid: string) {
    return ["identities", userUid];
}

/**
 * Transform the Firebase User into a SessionUser.
 */
export function transformUser(event: UserChangeEvent) {
    const api = event.api;
    const user = event.user;
    const leasee = event.leasee;
    const path = identityPath(user.uid);
    
    const [identity, identityError] = watchEntity(api, leasee, path, IDENTITY_OPTIONS);

    if (identity) {
        if (identity.displayName !== user.displayName) {
            updateDisplayName(api, user);
        }
        return createSessionUser(user, identity);
    } else if (identityError) {
        const message = "Cannot create SessionUser because an error occurred while loading the Identity";
        throw new Error(message, {cause: identityError});
    } else if (identity===null) {
        const message = "Cannot create SessionUser because the Identity document was deleted";
        throw new Error(message);
    } else {
        // The Identity record must be pending. The SessionUser entity will
        // be created and added to the cache by `identityTransform`.
        // For now, we return `undefined` to signal that the SessionUser entity is
        // pending.
        return undefined;
    }
}
```
### Use the api to transform the Firebase User into the enriched user

We need to modify the `FirebaseAuthListener` component to use the `transformUser` function.
Here's the revised component:
```tsx
// File: ./src/components/FirebaseAuthListener.tsx

import { useAuthLisener, AuthErrorEvent } from "@gmcfall/react-firebase-state";
import { alertError, alertSuccess } from "../Alert/alertApi";
import { transformUser } from "../../shared/identity"; // NEW

function handleError(event: AuthErrorEvent) {
    const api = event.api;
    const error = event.error;
    alertError(api, "An error occurred while getting information about your account", error);
}

interface FirebaseAuthListenerProps {
    children?: React.ReactNode;
}

export default function FirebaseAuthListener(props: FirebaseAuthListenerProps) {
    const {children} = props;
    useAuthListener({
        onError: handleError, 
        transform: transformUser // NEW
    });

    return (
        <>
            {children}
        </>
    )
}
```
The new lines of code are marked with a "NEW" comment.

## Get information about the current user
The `FirebaseAuthListener` component starts a listener that monitors changes 
to the state of the current user.  The following example shows how a component 
nested within the `FirebaseAuthListener` can get information about the current user.


```tsx
// File: ./src/components/CurrentUserIdentity.jsx

import { useAuthUser } from "@gmcfall/react-firebase-state";
import { SessionUser } from "../shared/types";
import { UserIdentity } from "./UserIdentity";

export function CurrentUserIdentity() {
    
    const [, user] = useAuthUser<SessionUser>();

    return (
        user ? (
            <UserIdentity 
                displayName={user.displayName} 
                handle={user.handle}
            />
        ) : null;
    )
}
```
The `useAuthUser` hook has a template parameter that allows you to specify the
type of the current user. In this example, we have `SessionUser` as the value
of the template parameter.

If you have not enriched the user entity with additional properties, you can omit
this parameter. The `useAuthUser` hook returns the Firebase `User` by default.

For completeness, here's the definition of the `UserIdentity` component:
```tsx
// File: ./src/components/UserIdentity.tsx

interface UserIdentityProps {
    displayName: string;
    handle: string;
}

export function UserIdentity(props: UserIdentityProps) {
    const {displayName, handle} = props;

    return (
        <span>{handle} ({displayName})</span>
    )
}
```
## Get information about any user

As we discussed earlier, you can always get information about users other than the 
current user by creating a Firebase Function that leverages the Admin SDK.

However, if you maintain a Firestore collection that replicates (and possibly enriches)
user data, then it is easier to fetch that data as illustrated by the following component.

```tsx
// File: ./src/components/AnyUserIdentity.tsx

import { useDocListener } from "@gmcfall/react-firebase-state";
import { Identity } from "../shared/types";
import { identityPath, IDENTITY_OPTIONS} from "../shared/identity";
import { UserIdentity } from "./UserIdentity";

interface AnyUserIdentityProps {
    userUid: string;
}

export function AnyUserIdentity(props: AnyUserIdentityProps) {
    const {userUid} = props;
    const path = identityPath(userUid);
    const [, identity] = useDocListener("AnyUserIdentity", path, IDENTITY_OPTIONS);

    return (
        identity ? (
            <UserIdentity
                displayName={identity.displayName}
                handle={identity.handle}
            />
        ) : null;
    )
}
```

----
[Listen for changes to the state of the current user]: #listen-for-changes-to-the-state-of-the-current-user
[Customize the properties of the user]: #customize-the-properties-of-the-user
[Define a Firebase collection that stores the extra user properties]: #define-a-firebase-collection-that-stores-the-extra-user-properties
[Define an interface for the enriched user]: #define-an-interface-for-the-enriched-user
[Define an api for managing the enriched user data]: #define-an-api-for-managing-the-enriched-user-data
[Use the api to transform the Firebase User into the enriched user]: #use-the-api-to-transform-the-firebase-user-into-the-enriched-user
[Get information about the current user]: #get-information-about-the-current-user
[Get information about any user]: #get-information-about-any-user
[Document Listeners]: ./document_listeners.html
[Manage client-side state]: ./manage_client_side_state.html