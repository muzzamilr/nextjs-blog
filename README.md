In this article, we will discuss how we integrate Redux store with our next js app by covering simple JWT auth and we will do all that without compromising on SSR.

Disclaimer: This is strictly a hands-on guide targeted towards developers with an advanced skill set in next js.

Following concepts will be covered.

* How to structure domain logic in Redux.
* Integrating Redux store with next js.
* Rebuilding state on page changes.
* Persisting state in cookies without sacrificing SSR.
* Setup server-side routing to support cookies.

Lets Code
Create the Next app using the command
```ts
npx create-next-app@latest
```

Now you can start the app with command using
```ts
npm run dev
```
This command will start the dev server at http://localhost:3000.

Looking towards the folder structure, base app includes the following directories
* pages
* public
* styles

Pages directory includes 'api' folder in which you can create your REST API.

Now you need to create the following directories in the root directory of the project, which includes,
* components
* store
* utils

After creating all these directories we need to install Redux-Toolkit using this command,
```ts
npm install @reduxjs/toolkit
```

First of all we need to create a file named auth-slice.js in store/auth directory
```ts
import { createSlice } from "@reduxjs/toolkit";

export const authSlice = createSlice({
  name: "auth",
  initialState = {
       isLoggedIn: false,
       user: {}
   };
  reducers: {
    setUser: (state, { payload }) => {
      state.user = payload.user;
    },
    deAuthenticate: (state) => {
      state.isLoggedIn = false
    },
    authenticate: (state, { payload }) => {
      state.isLoggedIn = true,
      state.user = payload
    },
    restoreAuthState: (state, { payload }) => {
      state.isLoggedIn = true
      state.user = payload
    }
  },
});

export const { setUser } = authSlice.actions;

export default authSlice.reducer;
```

Now we need to create a file called action-creators.js in auth directory,


Now, we need to configure our store and create a file named store.js in store directory,
```ts
import auth from "./auth/auth-slice";

const store = configureStore({
  reducer: auth,
});

export default store;
```

In _app.js file add the following code,
```ts
import '../styles/globals.css'
import { Provider } from "react-redux";
import store from "../store/store";

export default function App({ Component, pageProps }) {
  return (
  <Provider store={store} >
    <Component {...pageProps} />
  <Provider />
  )
}
```
