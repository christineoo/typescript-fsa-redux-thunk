# [TypeScript FSA](https://github.com/aikoven/typescript-fsa) utilities for redux-thunk
[![npm (tag)](https://img.shields.io/npm/v/typescript-fsa-redux-thunk/beta.svg)](https://github.com/xdave/typescript-fsa-redux-thunk)
[![npm](https://img.shields.io/npm/l/typescript-fsa-redux-thunk.svg)](https://github.com/xdave/typescript-fsa-redux-thunk/blob/v2/LICENSE.md)
[![GitHub last commit (branch)](https://img.shields.io/github/last-commit/xdave/typescript-fsa-redux-thunk/v2.svg)](https://github.com/xdave/typescript-fsa-redux-thunk)
[![Build Status][travis-image]][travis-url]
[![codecov](https://codecov.io/gh/xdave/typescript-fsa-redux-thunk/branch/v2/graph/badge.svg)](https://codecov.io/gh/xdave/typescript-fsa-redux-thunk)

### NOTE: There's probably breaking changes from 1.x.  Read on to find out more and check the notes at the bottom for more info.

## Installation
```
npm i typescript-fsa-redux-thunk@beta redux redux-thunk
```

## API

### `thunkToAction(ThunkActionCreator): ((Params) => Result)`
Another useful cast function that can help when attempting to extract the return
value out of your async action creator.  If the action is being pre-bound to
dispatch, then all we want back is the return value (the action object).
Coming soon: an example.  TL;DR: pass your async action creator into this before
passing it to `bindActionCreators` or the `mapDispatchToProps` object (react-redux).

### `bindThunkAction(AsyncActionCreators, AsyncWorker): ThunkActionCreator`
Creates redux-thunk that wraps the target async actions.
Resulting thunk dispatches `started` action once it is started and
`done`/`failed` upon finish.

### `asyncFactory<State, Extra>(ActionCreatorFactory): ((type: string, AsyncWorker) => ({ async: AsyncActionCreators, action: ThunkActionCreator }))`
Factory function to easily create a typescript-fsa redux thunk.

### * Of this API, you may only end up using `asyncFactory` and `thunkToAction` (if you're using `react-redux`)

**Example**
```ts
import 'isomorphic-fetch';
import { createStore, applyMiddleware, AnyAction } from 'redux';
import thunkMiddleware, { ThunkMiddleware } from 'redux-thunk';
import { reducerWithInitialState } from 'typescript-fsa-reducers';
import actionCreatorFactory from 'typescript-fsa';
import { asyncFactory } from 'typescript-fsa-redux-thunk';

/** Parameters used for logging in */
interface LoginParams {
	email: string;
	password: string;
}

/** The object that comes back from the server on successful login */
interface UserToken {
	token: string;
}

/** The shape of our Redux store's state */
interface State {
	title: string;
	userToken: UserToken;
	loggingIn?: boolean;
	error?: Error;
}

/** The typescript-fsa action creator factory function */
const create = actionCreatorFactory('examples');

/** The typescript-fsa-redux-thunk async action creator factory function */
const createAsync = asyncFactory<State>(create);

/** Normal synchronous action */
const changeTitle = create<string>('Change the title');

/** The asynchronous login action */
const login = createAsync<LoginParams, UserToken>(
	'Login',
	async (params, dispatch) => {
		const url = `https://reqres.in/api/login`;
		const options: RequestInit = {
			method: 'POST',
			body: JSON.stringify(params),
			headers: {
				'Content-Type': 'application/json; charset=utf-8'
			}
		};
		const res = await fetch(url, options);
		if (!res.ok) {
			throw new Error(
				`Error ${res.status}: ${res.statusText} ${await res.text()}`);
		}

		dispatch(changeTitle('You are logged-in'));

		return res.json();
	}
);

/** An initial value for the application state */
const initial: State = {
	title: 'Please login',
	userToken: {
		token: ''
	}
};

/** Reducer, handling updates to indicate logging-in status/error */
const reducer = reducerWithInitialState(initial)
	.case(changeTitle, (state, title) => ({
		...state,
		title
	}))
	.case(login.async.started, state => ({
		...state,
		loggingIn: true,
		error: undefined
	}))
	.case(login.async.failed, (state, { error }) => ({
		...state,
		loggingIn: false,
		error
	}))
	.case(login.async.done, (state, { result: userToken }) => ({
		...state,
		userToken,
		loggingIn: false,
		error: undefined
	}));

/** Putting it all together */
(async () => {
	// Declaring the type of the redux-thunk middleware is what makes
	// `store.dispatch` work. (redux@4.x, redux-thunk@2.3.x)
	const thunk: ThunkMiddleware<State, AnyAction> = thunkMiddleware;
	const store = createStore(reducer, applyMiddleware(thunk));
	console.log(store.getState().title);

	try {
		const userToken = await store.dispatch(login.action({
			email: 'test@example.com',
			password: 'password'
		}));

		console.log(store.getState().title, userToken);
	} catch (err) {
		console.log(err);
	}
})();

```

Note: A change from 1.x is the result type is not always assumed to be a
Promise. If you want the result to be a promise, just return one from your
worker function; but continue to specify the result as `T` rather than
`Promise<T>` (same as 1.x).

[travis-image]: https://travis-ci.org/xdave/typescript-fsa-redux-thunk.svg?branch=v2
[travis-url]: https://travis-ci.org/xdave/typescript-fsa-redux-thunk
