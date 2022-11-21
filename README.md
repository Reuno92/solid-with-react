# SOLID Principe wih react

> Author: [Mohammad Faisal](https://www.mohammadfaisal.dev/) - his [Website](https://www.mohammadfaisal.dev/)
>
> Original Post:
>  * [Single Responsibility Principe](https://medium.com/better-programming/how-to-apply-solid-principles-to-clean-your-code-in-react-cdfd5e0a9cea)
>  * [Open Close Principe](https://betterprogramming.pub/applying-the-open-closed-principle-to-write-clean-react-components-4e4514963e40)
>  * [Liskov Substitution Principe](https://betterprogramming.pub/applying-the-liskov-substitution-principle-in-react-3a0614a42a08)
>  * [Interface Segregation Principe](https://betterprogramming.pub/how-to-apply-interface-segregation-principle-in-reactjs-fadf77113c5d)
>  * [Dependency Inversion Principe](https://betterprogramming.pub/apply-the-dependency-inversion-principle-in-react-c20a0afc3d64)
>
>  Rewriter: Renaud Racinet

## How To Apply SOLID Principles To Clean Your Code in React

> A look at the single-responsibility principle (SRP) in action

The main purpose of SOLID principles is to work as a guideline for software professionals who care about their craft.
People who take pride in building beautifully designed code bases that stand the test of time.

### Single Responsibility Principe

For example

#### 1. Move Data Processing Logic Out

Never keep your HTTP calls inside the component. It’s a rule of thumb. There are several strategies you can follow to
remove these codes from the component.

The least you should do is create a custom Hook and move your data-fetching logic there. For example, we can create a
Hook named useGetRemoteData that looks like this:

*useGetRemoteData.ts*

```typescript
import {useEffect, useReducer, useState, FC} from "react";

const initialState: { isLoading: boolean } = {
    isLoading: true,
};

function reducer(state: { isLoading: boolean }, action: { type: string, payload: unknown }) {
    switch (action.type) {
        case 'LOADING':
            return {isLoading: true};
        case 'FINISHED':
            return {isLoading: false};
        default:
            return state;
    }
}

export const useGetRemoteData: FC = (url) => {

    const [users, setUsers] = useState<UserModel>([])
    const [state, dispatch] = useReducer(reducer, initialState);

    const [filteredUsers, setFilteredUsers] = useState<Array<UserModel>>([])


    useEffect(() => {
        dispatch({type: 'LOADING'})
        fetch('https://jsonplaceholder.typicode.com/users')
            .then(response => response.json())
            .then(json => {
                dispatch({type: 'FINISHED'})
                setUsers(json)
            })
    }, [])

    useEffect(() => {
        const filteredUsers = users.map(user => {
            return {
                id: user.id,
                name: user.name,
                contact: `${user.phone} , ${user.email}`
            };
        });
        setFilteredUsers(filteredUsers)
    }, [users])

    return {filteredUsers, isLoading: state.isLoading}
}
```

> **Rewriter notes**
>
> Initial loading can be union type *pending*, *success*, and *error* for more comprehension and Readability.
>
> And if you do that, it preferable to create a data model for your fetch.
>
> Props url is useless, it isn't used.

#### 2. Reusable Hook for Data Fetching

Now when we see our `useGetRemoteData` Hook, we see that this Hook is doing two things:

1. Fetching data from a remote source
2. Filtering data

Let's extract the logic of fetching remote data to a separate Hook named `useHttpGetRequest` that takes the URL as a
component:

*useHttpGetRequest.ts*

```typescript
import {useEffect, useReducer, useState} from "react";
import {loadingReducer} from "./LoadingReducer";

const initialState: { isLoading: boolean } = {
    isLoading: true
};

export const useHttpGetRequest: FC<{ URL: string }> = (URL: string) => {

    const [users, setUsers] = useState<Array<UserModel>>([])
    const [state, dispatch] = useReducer(loadingReducer, initialState);

    useEffect(() => {
        dispatch({type: 'LOADING'})
        fetch(URL)
            .then(response => response.json())
            .then(json => {
                dispatch({type: 'FINISHED'})
                setUsers(json)
            })
    }, [])

    return {users, isLoading: state.isLoading}

}
```

>
> **Rewriter notes:**
> Can be use a generic Type for more re-usability
> replace `useHttpGetRequest` by `useHttpGetRequest<T>`
> after change state user with:
> ```typescript 
> const [users, setUsers] = useState<Array<UserModel>([]);
> ```
> by
> ```typescript
> const [data, setData] = useState<T|null>(null);
> ```
> It's clearly recommend to use environment variable into this customHook, for take a base url if you consume only one
> API REST.

We also removed the reducer logic to a separate file:

*loadingReducer.ts*

```typescript
export function loadingReducer(state: { isLoading: boolean }, action: { type: string, payload: unknown }) {
    switch (action.type) {
        case 'LOADING':
            return {isLoading: true};
        case 'FINISHED':
            return {isLoading: false};
        default:
            return state;
    }
}
```

So now our `useGetRemoteData` becomes:

*useGetRemote.ts*

```typescript
import {useEffect, useState, FC} from "react";
import {useHttpGetRequest} from "./useHttpGet";

const REMOTE_URL = 'https://jsonplaceholder.typicode.com/users'

export const useGetRemoteData: FC = () => {
    const {users, isLoading} = useHttpGetRequest(REMOTE_URL)
    const [filteredUsers, setFilteredUsers] = useState([])

    useEffect(() => {
        const filteredUsers = users.map(user => {
            return {
                id: user.id,
                name: user.name,
                contact: `${user.phone} , ${user.email}`
            };
        });
        setFilteredUsers(filteredUsers)
    }, [users])

    return {filteredUsers, isLoading}
}
```

Much cleaner, right? Can we do better? Sure, why not?

#### 3. Decompose UI Components

Take a look at our component where we show the details of a user. We can create a reusable `UserDetails` component for
that purpose:

```typescript jsx
const UserDetails: FC<{ user: UserModel }> = (user: UserModel) => {

    const showDetails = (user: UserModel) => {
        alert(user.contact)
    }

    return (
        <ul key={user.id} onClick={() => showDetails(user)}>
            <li>{user.name}</li>
            <li>{user.email}</li>
        </ul>
    )
}
```

Finally, our original component becomes:

```typescript jsx
import React, {FC} from "react";
import {useGetRemoteData} from "./useGetRemoteData";

export const Users: FC = () => {
    const {filteredUsers, isLoading} = useGetRemoteData<{ filteredUsers: Array<UserModel>, isLoading: boolean }>()

    return (
        <>
            <h2>Users List</h2>
            <div> Loading state: {isLoading ? 'Loading' : 'Success'}</div>
            {
                filteredUsers.map(
                    user => (
                        <UserDetails user={user}/>
                    )
                )
            }
        </>
    );
}
```

We slimmed our code down from 60 lines to 19 lines! And we created five separate components, each with a clear and
single responsibility.

#### Let's Review What We Just Did

Let’s review our components and see if we achieved the SRP:

* `Users.js` — Responsible for displaying the user list
* `UserDetails.js` — Responsible for displaying details of a user
* `useGetRemoteData.js`  — Responsible for filtering remote data
* `useHttpGetrequest.js` — Responsible for HTTP calls
* `LoadingReducer.js` — Complex state management

Of course, we can improve a lot of other things, but this should work as a good starting point for you.

#### Conclusion

This was a simple demonstration of how you can reduce the amount of code in each file and create beautiful and reusable
components with the power of SOLID.

### Applying the Open-Closed Principle To Write Clean React Components

> A look at the SOLID principles in action

SOLID is a set of principles. They are mainly guidelines for software professionals who care about their code quality
and maintainability.

React is not object-oriented by nature, but the main ideas behind these principles can be helpful. In this article, I
will try to demonstrate how we can apply these principles to write better code.

In a [previous chapter](#single-responsibility-principe), we talked about the single-responsibility principle. Today, we
will discuss the second principle of SOLID: the Open-Closed principle.

What Is the Open-Closed Principle?

According to [Thorben Janssen on Stackify](https://stackify.com/solid-design-open-closed-principle/):

> “Robert C. Martin considered this principle as the ‘most important principle of object-oriented design.’ But he wasn’t
> the first one who defined it. Bertrand Meyer wrote about it in 1988 in his book Object-Oriented Software Construction.
> He explained the Open/Closed principle as:<br><br> ‘Software entities (classes, modules, functions, etc.) should be open
> for extension, but closed for modification.’”

This principle tells you to write code in such a way that you will be able to add additional functionality without
changing the existing code.

Let’s see where we can apply this principle.

#### Let’s Start With an Example

Say we have a `User` component where we pass a user's details and the main purpose of this class is to show the details
of that particular user.

```typescript jsx
import React from 'react';

export const User: FC<UserModel> = ({user}) => {

    return <>
        <li> Name: {user.name}</li>
        <li> Email: {user.email}</li>
    </>
}
```

This is simple enough to start with. But our life is not so simple. After a few days, our manager tells us that there
are three types of users in our system: `SuperAdmin`, `Admin`, etc.

And each of them will have different information and functionalities.

#### What’s The Solution?

Now we need to design our code in such a way that we don’t need to add a conditional inside the User.js component. Let’s
create a separate component for SuperAdmin:OK, so there are two main techniques that we can apply in this scenario:

    * Higher-order component
    * Component composition

It’s better to go the second route whenever possible, but there can be cases where using a HOC is necessary.

For now, we will use a technique recommended by Facebook that is called the composition of components.

#### Let’s Create Separate User Components

Now we need to design our code in such a way that we don’t need to add a conditional inside the `User.js` component.
Let’s create a separate component for `SuperAdmin`:

```typescript jsx
import React, {FC} from 'react';
import UserModel from './model/User.model';
import {user} from "./User";

export const SuperAdmin: FC<{ user: UserModel }> = ({user: UserModel}) => {

    return <>
        <User user={user}/>
        <div> This is super admin user details</div>
    </>
}
```

Similarly, another one for Admin users:

```typescript jsx
import React from 'react';
import UserModel from './model/User.model';
import {user} from "./User";

export const Admin: FC<{ user: UserModel }> = ({user: UserModel}) => {

    return (
        <>
            <User user={user}/>
            <div> This is admin user details</div>
        </>
    )
}
```

And now our App.js file becomes:

```typescript jsx
import React from 'react';
import Admin from './Admin';
import SuperAdmin from './SuperAdmin';

export default function App() {
    const user = {};

    const userByTypes = {
        'admin': <Admin/>,
        'superadmin': <SuperAdmin/>
    };

    return (
        <div>
            {
                userByTypes[`${user.type}`]
            }
        </div>
    )
}
```

Now we can create as many user types as we need. Our logic for particular users is encapsulated, and we don’t need to
revisit our code for any additional modifications.

Some might argue we are increasing the number of files unnecessarily.
Sure, you can leave it as-is for now, but you will definitely feel the pain as the complexity of the application grows.

##### Caution

SOLID is a set of principles. They are not mandatory for you to apply in every scenario. As a seasoned developer, you
should find a good balance between code length and readability.

Don’t obsess too much over these principles. In fact, there is a famous phrase to explain these scenarios:

> “Too Much SOLID.”

So knowing these principles is good, but you have to keep a balance. You may not need these compositions for one or two
extra fields, but keeping them separate will definitely help in the long run.

#### Conclusion

Knowing these principles will take you a long way because at the end of the day, a good piece of code is what matters
and there is no single way of doing things.

The third principle of SOLID (Liskov Substitution Principle) is
discussed [here](https://betterprogramming.pub/applying-the-liskov-substitution-principle-in-react-3a0614a42a08)

Resource: [Open Close Principe](https://stackify.com/solid-design-open-closed-principle/)

### What Is the Liskov Substitution Principle?

In simple terms, this principle says:

> “Subclasses should be substitutable for their superclasses.”

That means subclasses of a particular class should be able to replace the superclass without breaking any functionality.

Example:
If `PlasticDuck` is a subclass of `Duck`, then we should be able to replace instances of `Duck` with `PlasticDuck`
without any surprises.

![](./img/duckandplasticduck.webp)

That means `PlasticDuck` should fulfill all the expectations set by the `Duck` class.

### What Does This Mean in React?

React is not an object-oriented framework because it’s basically JavaScript. In the context of React, the main idea
behind this principle is:

> “Components should abide by some kind of contract.”

At its core, this means there should be some kind of contract between components. So whenever a component uses another
component, it shouldn’t break its functionality (or create any surprises).

### Let's Take a Deeper Dive

Let’s take a `ModalHolder` component. This component takes `contentToShow` as a prop and shows it inside a modal:

```typescript jsx
import {useState} from "react";
import Modal from 'react-modal';

export const ModalHolder: FC<PropWithChildren> = ({contentToShow}) => {

    const [visibility, setVisibility] = useState(false);

    return <>
        <button onClick={() => setVisibility(true)}> Show Modal</button>

        <Modal isOpen={visibility}>
            <div>{contentToShow}</div>
        </Modal>
    </>
}
```

What’s the issue here?

Well, the problem is now there are no restrictions on what can be passed into the `ModalHolder` component. Absolutely
anything can be passed into this through the variable `contentToShow`.

First, let’s check if our code works and everything goes as expected:

```typescript jsx
import React, {useEffect} from 'react';
import {ModalHolder} from "./views/liskov-substitution-principle/ModalHolder";

function App() {

    const modalContent = (<div> This is shown inside modal </div>);

    return (
        <div>
            <ModalHolder contentToShow={modalContent}/>
        </div>
    );
}

export default App;
```

Now if you open the modal, it will work just fine and show you the modal:

Let’s take advantage of the flaw we described earlier and see how it can destroy our application.

Let's try to pass an object into the `ModalHolder`  and see what happens:

```typescript jsx
import React, {useEffect} from 'react';
import {ModalHolder} from "./views/liskov-substitution-principle/ModalHolder";

function App() {

    const modalContent = {key: " value"}

    return (
        <div>
            <ModalHolder contentToShow={modalContent}/>
        </div>
    );
}

export default App;
```

This code is perfectly fine and will give no compilation error. Now let's open our application and see what happens if
we click on the button:

![](https://miro.medium.com/max/1100/1*67ENRHAiKBwYpndTSVFT-w.webp);

So our application is crashing even though our code has no error. What went wrong here?

Our `Modal` component is allowed to contain another React component. But other components are not bonded to follow that
because there is no contract.

### What’s the Solution?

Now we will see the importance of using TypeScript in our application and why it’s important. Let's refactor
our `ModalHolder` component to TypeScript and see what happens:

```typescript jsx
import {ReactElement, useState} from 'react';
import Modal from 'react-modal';

interface ModalHolderProps {
    contentToShow: JSX.Element
}

export const ModalHolder = ({contentToShow}: ModalHolderProps) => {
    const [visibility, setVisibility] = useState(false)

    return (
        <>
            <button onClick={() => setVisibility(true)}> Show Modal</button>

            <Modal isOpen={visibility}>
                <div>{contentToShow}</div>
            </Modal>
        </>
    )
}
```

So now we have refactored our component to accept the prop `contentToShow` only when it gets a `JSX.Element`.

If someone wants to pass anything that’s not a valid component to render, we will get an error:

![](https://miro.medium.com/max/1100/1*w4d0iPlF-h1CFJkWHSkiIg.webp)

Voilà! Now all other components that want to plug into the ModalHolder component need to follow a contract so that they
don’t create any unexpected behavior.

#### Did We Do It?

We have designed our `ModalHolder` component in such a way that no child component that uses this component is able to
create any unexpected behavior because they must abide by the rules set by the parent.
That’s exactly what **Liskov Substitution Principle** is all about.

So yes, we did it!

### How to Apply Interface Segregation Principle in ReactJS

SOLID is a set of principles that are not specific to any framework or language. These principles help us to understand
how to write beautiful applications for our customers.

Today we will talk about the fourth principle of SOLID:

#### Interface Segregation Principle

We will try to understand the underlying concept of this principle and implement it in the context of **ReactJS**.

#### What’s This Principle All About?

According to [Stackify](https://stackify.com/interface-segregation-principle/), the Interface Segregation Principle says

> Clients should not be forced to depend upon interfaces that they do not use

In ReactJS, **we don’t use any interface**, at least not in the sense of object-oriented programming. So the main
takeaway for this scenario is:

> Components should not depend on things they don’t need.

Now, let’s see how this principle can help us to write clean and beautiful ReactJS components.

#### A Practical Approach

Let’s say we have a `User` component that’s responsible for displaying the details of a user. Our user object looks
something like this:

*User.ts*

```typescript
export interface User {
    name: "Some user",
    age: "60",
    adderess: "House Address",
    bankName: "Some Bank",
    bankAccountNumber: "1234567890"
}
```

However, it uses two children components, named `PersonalDetails` and `BankingDetails` to show the details.

Our `User` component looks something like this:

*User.tsx*

```typescript jsx
import React from "react";
import User from '../interface/user.interface';

export const User: FC<{ user: User }> = ({user: User}) => {

    return (
        <>
            <PersonalDetails user={user}/>
            <BankingDetails user={user}/>
        </>
    )
}
```

Similarly, our `PersonalDetails` looks like this:

*PersonalDetails.tsx*

```typescript jsx
import React, {FC} from "react";
import User from '../interface/user.interface';

export const PersonalDetails: FC<{ user: User }> = ({user}: User) => {

    return (
        <>
            <li>name: {user.name}</li>
            <li>name: {user.age}</li>
            <li>name: {user.address}</li>
        </>
    );
}
```

And our BankingDetails.js looks like this:

*BankingDetails.tsx*

```typescript jsx
import React from "react";
import User from '../interface/user.interface';

export const BankingDetails: FC<{ user: User }> = ({user}: User) => {

    return (
        <>
            <li>Bank Name: {user.bankName}</li>
            <li>Bank Account Name: {user.bankAccountNumber}</li>
        </>
    )

}
```

So, what’s the problem with this approach? Well, two things.

Firstly, our `PersonalDetails` component doesn't need banking information and our `BankingDetails` component doesn't
need personal details to function, so it’s clearly violating the `Interface Segregation Principle`.

Secondly, In the future, if we want to add typescript to our project (which you should) then to test PersonalDetails
you’ll be required to mock the whole user object, though the banking information has nothing to do with PersonalDetails.

> His approach is good but I dislike the manner.

There are several approaches for fixing this problem, but the underlying principle is the same.

> We need to pass only the relevant information to the children components.

So we will break down our data object and pass the appropriate parts to the respective components only.

![](https://miro.medium.com/max/1100/1*uTVgTG3lCHIfK3ZNOrwL2A.png)
*[Image Credit](https://medium.com/@learnstuff.io/interface-segregation-principle-dd885e59aec9)*

Something like this:

*User.Model.ts*

```typescript
export default class User {
    constructor(
        public personalDetails: PersonalDetails,
        public bankingDetails: Banking,
    ) {
    }
}

export interface PersonalDetails {
    name: string,
    age: number,
    address: string,
}

export interface BankingDetails {
    bankName: string,
    bankAccountName: number,
}
```

And the `User Component`:

*User.tsx*

```typescript jsx
export const User: FC<{ user: User }> = ({user: User}) => {
    return (
        <>
            <PersonalDetails user={user.personalDetails}/>
            <BankingDetails user={user.bankingDetails}/>
        </>
    )
}
```

We’ve broken our user data into two parts with new keys `personalDetails` and `bankingDetails` and we passed this
specific piece of data to our child components.

#### The Second Approach

The previous solution is perfect, but what if you don’t have control over the data? Perhaps it’s fetched from a remote
source. Or what if you don’t want to modify the data structure for some reason?

Don’t worry. We can apply another technique to solve this:

*User.model.ts*

```typescript
export default class User implements PersonalDetails, BankingDetails {

    constructor(
        public name: string,
        public age: number,
        public address: string,
        public bankName: string,
        public bankAccountName: number
    ) {
    }
}

export interface PersonalDetails {
    name: string,
    age: number,
    address: string,
}

export interface BankingDetails {
    bankName: string,
    bankAccountName: number,
}
```

*User.tsx*

```typescript jsx
import User from '../abstract/User.model';

export const User: FC<User> = (props) => {

    const {name, age, address, bankName, bankAccountNumber} = props;

    return (
        <>
            <PersonalDetails name={name} age={age} address={address}/>
            <BankingDetails bankName={bankName} bankAccountNumber={bankAccountNumber}/>
        </>
    )
}
```

In our child components, we can use this as the following:

*PersonaDetails.tsx*

```typescript jsx
import React, {FC} from "react";
import {BankDetails} from '../abstract/User.model'; // Don't forget to use export before your

export const BankingDetails: FC<BankDetails> = ({bankName, bankAccountNumber}: BankDetails) => {

    return (
        <>
            <div>bankName: {bankName}</div>
            <div>bankAccountNumber: {bankAccountNumber}</div>
        </>
    )
}
```

*BankDetails.tsx*

```typescript jsx
import React, {FC} from "react";
import {PersonalDetails} from '../absract/User.model';

export const PersonalDetails: FC<PersonalDetails> = ({name, age, address}: PersonalDetails) => {
    return (
        <>
            <div>name: {name}</div>
            <div>age: {age}</div>
            <div>address: {address}</div>
        </>
    )
}
```

Now we are forced to pass only the relevant data to the children components. No more unwanted bugs for you!

#### Final Thoughts

These are just principles to guide your way of thinking — not hard and fast rules to build your application.

Knowledge of these concepts will take you ahead of others. These concepts will surely help you to understand the core
principles of programming.

> Because frameworks are temporary but concepts are permanent

## Apply the Dependency Inversion Principle in React

The *dependency inversion principle* is one of the famous SOLID principles. Also, it is one of the most important ones.

Today, we will see how to solve a very common mistake that novice React developers make using this principle.

I will try to keep it very simple. Let’s get started!

### What Does This Principle Tell Us?

In terms of object-oriented programming, the main idea behind this principle is to always have a high-level code
interface with abstraction rather than an implementation detail.

Hold on! I know what you are thinking: “I am a simple frontend developer. Why are you bothering me with these complex
terms?”

Let me state it simply for you. For a React application, this principle means:

> “No component or function should care about how a particular thing is done.”

Still not clear? OK, let’s get our hands dirty with some code!

### A Practical Example

Let’s take a very common use case. We are going to make an API call from our component to get some data from a remote
source. An implementation can look like this:

```typescript jsx
import React from "react";

const REMOTE_URL = 'https://jsonplaceholder.typicode.com/users'

export const Users: FC<UserModel> = () => {

    const [users, setUsers] = useState([])

    useEffect(() => {

        fetch(URL)
            .then(response => response.json())
            .then(json => setUsers(json))

    }, [])

    return <>
        <div> Users List</div>
        {
            filteredUsers.map(user =>
                (
                    <div>
                        {
                            user.name
                        }
                    </div>
                )
            )
        }
    </>
}
```

Look at this component. It depends on some remote data that is fetched right inside the component.

Our `Users` component’s main responsibility is to render the data. It should not care about how data is fetched or where
the data comes from.

This component knows too much — and that’s a problem.

### Why?

Well, let’s say you have ten other components and all of them fetch their own data.

Now your manager comes along and tells you to use `axios` instead of `fetch`
You are in trouble! Now you have to go into each file and refactor the logic to use `axios`.

But life is not so simple! After a few days, your manager comes again and tells you to implement caching.

You have to do the same thing once again.

Thus, it increases the chance of introducing a bug in your software. Also, the code becomes unmaintainable and valuable
time is wasted.

### So What Should We Do Then?

Let’s introduce a data-fetching Hook and abstract away our logic outside our component because that’s exactly what this
principle tells us. To depend on abstraction, remember?

*useFetch.ts*

```typescript
import {useState} from "react";

export const useFetch = (URL) => {

    const [data, setData] = useState([])

    useEffect(() => {

        fetch(URL)
            .then(response => response.json())
            .then(json => setData(json))

    }, [])

    return data;

}
```

Now use this Hook inside our Users component:

*Users.tsx*

```typescript jsx
import React from "react";
import useFetch from './useFetch'

const REMOTE_URL = 'https://jsonplaceholder.typicode.com/users'

export const Users = () => {

    const users = useFetch(REMOTE_URL)

    return <>
        <h1>Users List</h1>
        {filteredUsers.map(user => <div>{user.name}</div>)}
    </>
}
```

Notice a great thing about this solution: Your `useFetch` Hook doesn’t care about who is calling it. It just takes a `URL` as an input and returns the data.

Now all other components can take advantage of the Hook that we just wrote. And our `Users` component no longer depends on the concrete details on how the data is coming back or which library is being used!
