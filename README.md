In this article, we will discuss how we can integrate Redux store with our next js app by covering simple JWT auth and we will do all that without compromising on SSR.

Disclaimer: This is strictly a hands-on guide targeted towards developers with an advanced skill set in NextJS.

Here's a list of the concepts that will be covered in this blog:

* Structuring domain logic in Redux.
* Integrating Redux store with NextJS.
* Rebuilding state on page changes.
* Persisting state in cookies without sacrificing SSR.

Let's Code!

A Next app can be created with the following shell command
```ts
npx create-next-app@latest
```

The development server for the  app can be started by running the following script.
```ts
npm run dev
```
The dev server can now be accessed at http://localhost:3000.

Looking towards the folder structure, base app includes the following directories
* pages
* public
* styles

Pages directory includes the 'api' folder which you can use to create your REST API with Next.

Now you need to create the following directories in the root directory of the project
* components
* store


## Dependencies
Our dependency list for this example application is very brief. We'll be using the following dependencies.

* Redux-toolkit
* Jsonwebtoken
* Axios
* Cookies-next

After creating all these directories we need to install Redux-Toolkit and a few other packages that we'll be using,
Our dependencies for this example are listed below.
```sh
npm install @reduxjs/toolkit react-redux jsonwebtoken axios cookies-next
```

### Redux Toolkit Configuration
Redux toolkit massively simplifies the usage of Redux. It comes recommended as the standard way to write Redux logic.
It removes a lot of boilerplate that used to be necessary to get Redux up and running and it integrates very well with
React's functional components as it provides a few very useful hooks out of the box.

A slice in Redux is exactly what its name suggests: a slice of the globally accessible Redux store that is used to share application logic across all components of the application.

In our case, we will create an authentication slice that will contain the shared logic needed for authentication purposes in the application. By convention each slice is put into a separate directory so we'll create an auth directory for the authentication slice that will house our slice's logic. The logic here will be encapsulated inside a file named auth-slice.js.

### store/auth/auth-slice.js
```ts
import { createSlice } from '@reduxjs/toolkit';
import { getCookie } from 'cookies-next';

const initialState = getCookie('token') ? {
  isLoggedIn: true,
  user: {}
} : {
  isLoggedIn: false,
  user: {}
};

export const authSlice = createSlice({
  name: 'auth',
  initialState: initialState,
  reducers: {
    deAuthenticate: (state) => {
      state.isLoggedIn = false;
    },
    authenticate: (state, action) => {
      state.isLoggedIn = true;
      state.user = action.payload;
    },
    restoreAuthState: (state, action) => {
      state.isLoggedIn = true;
      state.user = action.payload;
    },
  },
});

export const { deAuthenticate, authenticate, restoreAuthState } =
  authSlice.actions;

export default authSlice.reducer;
```

As you can see, this file makes use of the createSlice function provided by Redux toolkit. We provide this function with an object that contains the name of our slice, its initial state and its reducers. Reducers are functions that transform our initial state into our desired state. You can see that we have directly mutated our initial state in these reducers but bear in mind that this is not what actually happens, the updates to the state are always applied immutably but all of that gets handled by Redux-toolkit itself. This saves us from having to write a lot of boilerplate code.

The initial state of the slice is declared dynamically by checking for a jwt cookie, if that exists we'll assume that the user is logged in and vice versa.

Now we need to create a file called action-creators.js in the auth directory.
These functions will be used to dispatch reducers which update the state of our application.


### store/auth/action-creators.js
```ts
import { deleteCookie } from 'cookies-next';
import { authenticate, deAuthenticate, restoreAuthState } from './auth-slice';

export const loginUser = (user) => async (dispatch) => {
  dispatch(authenticate(user));
};

export const logoutUser = (user) => async (dispatch) => {
  deleteCookie('token');
  dispatch(deAuthenticate(user));
};

export const checkLogin = (user) => async (dispatch) => {
  dispatch(restoreAuthState(user));
};

```

Now, we need to configure our store and create a file named store.js in store directory,

###  store/store.js

```ts
import auth from "./auth/auth-slice";

const store = configureStore({
  reducer: auth,
});

export default store;
```

In the _app.js file we will add the following code to make the store globally accessible to each of our application's components,

### pages/_app.js

```ts
import '../styles/globals.css';
import { Provider } from 'react-redux';
import store from '../store/store';
import Head from 'next/head';

export default function App({ Component, pageProps }) {
  return (
    <>
      <Head>
        <link
          href="//netdna.bootstrapcdn.com/bootstrap/4.5.2/css/bootstrap.min.css"
          rel="stylesheet"
        />
      </Head>
      <Provider store={store}>
        <Component {...pageProps} />
      </Provider>
    </>
  );
}
`````
We've also added a link to bootstrap's CDN as we'll be using it for this app's styling.

### pages/index.js

This is the "/" route of the Next.JS application. In this code we are using server side rendering to verify the JWT token and if the token is not verified then router pushes to the signin page.
getServerSideProps is invoked before the rendering of component and returns the props which are further used by component.

```ts
import { getCookie } from "cookies-next";
import jwt from "jsonwebtoken";
import Head from "next/head";
import { useRouter } from "next/router";
import { useEffect } from "react";
import { useDispatch, useSelector } from "react-redux";
import { logoutUser } from "../store/auth/action-creators";
import styles from "../styles/Home.module.css";

export const getServerSideProps = async ({ req, res }) => {
	const cookie = getCookie("token", { req, res });
	if (!cookie) return { props: { isAuthenticated: false } };

	try {
		const isAuthenticated = await jwt.verify(
			cookie,
			process.env.NEXT_PUBLIC_JWT_KEY,
		);
		return { props: { isAuthenticated: isAuthenticated } };
	} catch (err) {
		return { props: { isAuthenticated: false } };
	}
};

export default function Home({ isAuthenticated }) {
	const dispatch = useDispatch();
	const isLoggedIn = useSelector((state) => state.auth?.isLoggedIn);
	const router = useRouter();

	useEffect(() => {
		// rome-ignore lint/complexity/useSimplifiedLogicExpression: <explanation>
		if (!isAuthenticated || !isLoggedIn) {
			dispatch(logoutUser());
			router.push("/signin");
		}
	}, [isAuthenticated, isLoggedIn]);

	return (
		<div className={styles.container}>
			<Head>
				<title>Create Next App</title>
			</Head>
			<p>hello</p>

			<button onClick={() => dispatch(logoutUser())}>Logout</button>
		</div>
	);
}

```

## Login Workflow

For the login page we will create a file named signin.js in the pages directory.
NextJS automatically adds routes for pages that correspond to each file in this directory.

We have a simple login page for this app, it looks like this:

### pages/signin.js
```ts
import { useEffect } from 'react';
import { useDispatch } from 'react-redux';
import { Login } from '../components/login';
import { loginUser } from '../store/auth/action-creators';
import { useRouter } from 'next/router';
import { useSelector } from 'react-redux';
import axios from 'axios';
import { setCookie } from 'cookies-next';

const LoginPage = () => {
  const dispatch = useDispatch();
  const router = useRouter();
  const isLoggedIn = useSelector((state) => state.auth?.isLoggedIn);

  useEffect(() => {
    if (isLoggedIn) {
      router.push('/');
    }
  }, [isLoggedIn]);

  const handleSubmit = async (user) => {
    await axios.post('http://localhost:3000/api/auth/', user).then((res) => {
      if (res.status === 200) {
        setCookie('token', res.data);
        dispatch(loginUser(user));
      }
    }).catch((err) => console.log(err));
  };

  return (
    <div>
      <Login login={handleSubmit} />
    </div>
  );
};
export default LoginPage;
```

So in this page, we check from our redux store if the user is already logged in or not. If they are logged in, they get redirected to the home page of our app otherwise they can put their credentials in the form provided by the Login component.

A sign in request is sent to the server once the user submits their credentials.
The server responds to the request with a JWT (JSON web token) that we store in a client-side cookie. This provides us with a mechanism to keep the user logged in.
We'll discuss it in more detail in our section concerning the server-side code.

The login component provides a simple form where the user may enter their credentials.

### components/login.js
```ts
import { useState } from 'react';

export const Login = ({ login }) => {
  const [state, setState] = useState({
    username: null,
    password: null,
    rememberMe: false,
  });

  const handlePhoneNumChange = (e) => {
    setState({ ...state, username: e.target.value });
  };

  const handlePinCodeChange = (e) => {
    setState({ ...state, password: e.target.value });
  };

  const handleRememberMeCheck = (e) => {
    setState({ ...state, rememberMe: e.target.value });
  };

  const handleSubmit = async (event) => {
    event.preventDefault();
    if (state.password !== null && state.username !== null)
      login(state);
  };

  return (
    <main className="main-content">
      <div className="bg-white rounded shadow-7 w-400 mw-100 p-6">
        <h5 className="mb-7">Sign into your account</h5>

        <form id="login" onSubmit={handleSubmit}>
          <div className="form-group">
            <input
              onChange={handlePhoneNumChange}
              type="text"
              className="form-control"
              name="phone_number"
              placeholder="eg. +34645136228"
            />
          </div>

          <div className="form-group">
            <input
              onChange={handlePinCodeChange}
              type="password"
              className="form-control"
              name="customer_pin"
              placeholder="Enter your pin"
            />
          </div>

          <div className="form-group flexbox py-3">
            <div className="">
              <input
                type="checkbox"
                onChange={handleRememberMeCheck}
                className="remember"
              />
              <label className="remember">Remember me</label>
            </div>

            <a className="text-muted small-2" href="/reset">
              Forgot password?
            </a>
          </div>

          <div className="form-group">
            <button className="btn btn-block btn-primary" type="submit">
              Login
            </button>
          </div>
        </form>

        <hr className="w-30" />
        <p className="text-center text-muted small-2">
          Don't have an account? <a href="/register">Register here</a>
        </p>
      </div>
    </main>
  );
};
```

## API Routes

### api/auth.js

In this file, we have created a request handler, which takes a user information as request body and jwt.sign method is used to generate a token. The token is generated everytime when user logs in to the app. If the request method is not appropriate then it will return unauthorised status.

```ts
import jwt from "jsonwebtoken";

export default function handler(req, res) {
	if (req.method === "POST") {
		const user = {
			username: req.body.username,
			status: "got it",
		};
		const token = jwt.sign(
			{
				user: user,
			},
			process.env.NEXT_PUBLIC_JWT_KEY,
			{ expiresIn: "12h" },
		);

		return res.json(token);
	}

	return res.status(401).json("Invalid Request");
}
```

## Managing Next App
After successfully completing all these steps you will need to create a .env file which consists of a env variable and assign it a key (you can use any random combination you want to use)
```env
NEXT_PUBLIC_JWT_KEY = <Your Key>
```

Now you can run the app in development server using command,
```sh
npm run dev
```
In order to run this app in production environment you can use the following commands,
```sh
npm run build
```
```sh
npm run start
```

## Resources
Resources for further exploration of Redux toolkit and JSON Web Tokens
- [JWT Introduction](https://jwt.io/introduction)
- [Redux Toolkit Documentation](https://redux-toolkit.js.org/introduction/getting-started)
- [In-Depth Explanation of JWT](https://auth0.com/docs/secure/tokens/json-web-tokens)
