
# Black box testing React components connected to Redux

What's the better way to make your code more maintainable and easier to change in the future than writing tests?
Probably hiring a bunch of interns to test code manually after each commit, but we don't have time and money for that so tests will have to do.

In this post I will focus on the importance of black box testing and how this can be achieved in React.
I will be using `jest`, `react-testing-library`, `redux-thunk` and Typescript for my own sanity.  
I am also going to assume that you know the basics of these technologies.

In addition, I will only focus on Component Integration Testing - how to test flow of your component in isolated environment - because there are no good resources on this topic, and Unit Testing is well understood in the community.

## Black box testing, why bother?

I am big fan of black box testing, especially when working with front-end. In the world of *javaScript frameworks changing so fast that before you download npm dependencies this dependency is already outdated* I cannot imagine writing tests that can be easily broken by the change in implementation that doesn't have any impact on the behavior of the application.  
This kind of environment discourages developers from refactoring and writing tests because everything you touch has high chance of breaking something (be it tests or the actual code).

We can avoid this problem by changing the way of thinking about our code and trying to test *behavior* and not the internal implementation (that can be easily changed). Doing so greatly reduces the need of changing tests when we want to change the way that our code works (e.g. refactoring code for optimization reasons).

For example, when testing, let's say, spinner component for data loading, I shouldn't care if the way of triggering that spinner is implemented with events or some boolean flag, this is an *implementation detail* that can change, but the *behavior* of the spinner on that page will stay the same.

You might say, that this is a very basic example, real world is not that easy and components have some kind of dependencies (e.g. component is calling some endpoint to retrieve data), and how do I test that?

Keep that in mind that topic of this post is integration testing and how to do that in the most flexible way.  
Your tests suite should still consist mostly of unit tests, but from time to time integration tests can come in handy and I will present to you how to write integration tests that are cheap and easy to maintain.  

## Test scenario

*Note: All code samples can be found on my [github](https://github.com/Waksuu/react-exploratory/tree/after-redux-tests)*

Let's say that we want to test component that is responsible for managing movies, and for this basic example our component will have two methods: `retrieveMovies()` and `clearMovies()`.

`MoviePanel.component.tsx`

```typescript
type Props = LinkStateProps & LinkDispatchProps;

const MoviePanel: FC<Props> = (props: Props) => {
  useEffect(() => {
    props.retrieveMovies();
  }, []);

  return (
    <>
      <MovieList movies={props.movies} />
      <MovieControls clearMovies={props.clearMovies} />
    </>
  );
};

interface LinkStateProps {
  movies: MovieDTO[];
}

const mapStateToProps = (state: AppState): LinkStateProps => ({
  movies: state.movie.movies
});

interface LinkDispatchProps {
  retrieveMovies: () => Promise<void>;
  clearMovies: () => void;
}

const mapDispatchToProps = (
  dispatch: ThunkDispatch<AppState, undefined, AppActions>
): LinkDispatchProps => ({
  retrieveMovies: () => dispatch(retrieveMoviesAction()),
  clearMovies: () => dispatch(clearMovies())
});

export default connect(mapStateToProps, mapDispatchToProps)(MoviePanel);
```

`Movie.action.ts`

```typescript
export const retrieveMoviesAction = (): ThunkAction<Promise<void>,
    AppState,
    undefined,
    AppActions
> => async (dispatch: ThunkDispatch<AppState, undefined, AppActions>) => {
    const movies: MovieDTO[] = await getAllMoviesREST();
    dispatch(moviesRetrieved(movies));
};

const moviesRetrieved = (movies: MovieDTO[]): RetrieveMovieListAction => ({
    type: MovieListRetrieved,
    movies
});

export const clearMovies = (): ClearMovieListAction => ({  
  type: ClearMovieListRequest,  
});
```

`Movie.reducer.ts`

```typescript
const moviesInitialState: MovieDTO[] = [];

export const movieReducer = (
  state = moviesInitialState,
  action: MovieListActionTypes
): MovieDTO[] => {
  switch (action.type) {
    case MovieListRetrieved:
      return action.movies;
    case ClearMovieListRequest:
      return [];
    default:
      return state;
  }
};
```

Let's assume that method `getAllMoviesREST()`in `Movie.action.ts` is an API call that returns promise (for the simplicity of this example under the hood it is just a mock but I'll leave it to your imagination to do the REST).
Now there is one problem, how do we write test for a component that is depended on external API?
Well there are two most popular options:

You can intercept all API calls with some test interceptor (external library) but that will leave your test fragile and hard to mock up (any change in the API method will force you to do some changes in test) and we want to avoid that.  
Or we can apply *Inversion of Control* principle and take our dependency (the `getAllMoviesREST()` method) as a parameter to action (as a method reference) and then compose our component, with all of its dependencies in a `MoviePanel.component.tsx`.

## Making our component testable

Firstly let's make our action method (`retrieveMoviesAction()`in `Movie.action.ts`) independent of concrete implementation of `getAllMoviesREST()` by simply receiving this method as function parameter.

`Movie.action.ts`

```typescript
export const retrieveMoviesAction = (getAllMovies: () => Promise<MovieDTO[]>): ThunkAction<Promise<void>,
    AppState,
    undefined,
    AppActions
> => async (dispatch: ThunkDispatch<AppState, undefined, AppActions>) => {
    const movies: MovieDTO[] = await getAllMovies();
    dispatch(moviesRetrieved(movies));
};
```

Simple enough.

Ok but now our `MoviePanel.component.tsx` is complaining that function `retrieveMoviesAction()` requires 1 argument and not 0.

Now you might be tempted to just directly add concrete implementation of our `getAllMovies` method in `mapDispatchToProps` like this.

`MoviePanel.component.tsx`

```typescript
const mapDispatchToProps = (
  dispatch: ThunkDispatch<AppState, undefined, AppActions>
): LinkDispatchProps => ({
  retrieveMovies: () => dispatch(retrieveMoviesAction(getAllMoviesREST)),
  clearMovies: () => dispatch(clearMovies())
});
```

But this solution still blocks us from injecting mocks into our component.  
Now we need to apply *Inversion of Control* principle for `mapDispatchToProps` in conjunction with [currying](https://blog.bitsrc.io/understanding-currying-in-javascript-ceb2188c339).

```typescript
const mapDispatchToProps = (getAllMovies: () => Promise<MovieDTO[]>) => (
  dispatch: ThunkDispatch<AppState, undefined, AppActions>
): LinkDispatchProps => ({
  retrieveMovies: () => dispatch(retrieveMoviesAction(getAllMovies)),
  clearMovies: () => dispatch(clearMovies())
});

export default connect(
  mapStateToProps,
  (dispatch: ThunkDispatch<AppState, undefined, AppActions>) =>
    mapDispatchToProps(getAllMoviesREST)(dispatch)
)(MoviePanel);
```

Notice that we pass `getAllMoviesREST` in the connect function, allowing connect function to compose our component.

I will provide detailed explanation on how it exactly works [later](#how-are-we-able-to-inject-dependencies-in-connect-function).

In order to create MoviePanel component in tests we have to add few exports

`MoviePanel.component.tsx`

```typescript
export const MoviePanel: FC<Props> = (props: Props) => {
  // Actual content not important
};
```

```typescript
export  const  mapStateToProps = (state: AppState): LinkStateProps  => ({
  // Actual content not important
});
```

```typescript
export const mapDispatchToProps = (
  // Actual content not important
});
```

And in order to check if our component was changed upon some action we need to add `data-testid` in two places.

`MovieControls.component.tsx`

```typescript
export const MovieControls: FC<Props> = (props: Props) => {
  return (
    <button
      data-testid="clear-movies-button"
      onClick={props.clearMovies}
      className="my-button"
    >
      Clear movies!
    </button>
  );
};
```

`MovieList.component.tsx`

```typescript
export const MovieList: FC<Props> = (props: Props) => {
  return (
    <div data-testid="movie-list-component">
      {props.movies.map(movie => (
        <div key={movie.id}>{movie.name}</div>
      ))}
    </div>
  );
};
```

## Testing our component

Testing components connected to redux is very similar to testing typical components. The main difference is that we have to create component using `connect` function, wrap our component in `Provider` component and create mock store.

Let's start with creating our component with `connect` function.  

```typescript
import { mapStateToProps, mapDispatchToProps, MoviePanel } from  "../../../app/movie/MoviePanel.component";
...
const MoviePanelMock: FC = connect(mapStateToProps, (
            dispatch: ThunkDispatch<AppState, undefined, AppActions>
        ) => mapDispatchToProps(getAllMoviesMock)(dispatch))(MoviePanel);

const getAllMoviesMock = (): Promise<MovieDTO[]> => Promise.resolve(mockMovies);

const mockMovies: MovieDTO[] = [
    {
        id: 1,
        name: "Mock movie 1"
    },
    {
        id: 2,
        name: "Mock movie 2"
    },
]
```

(The component name has to start from upper case)  
Also note that we have to import `MoviePanel` as **not** default export (hence the curly brackets around import).  
And that's it, now we created mock component with injected dependencies.

Now we need to create store

```typescript
import { configureStore, AppState } from  "../../../common/redux/store/ConfigureStore";
...
const mockStore: Store<AppState, AppActions> = configureStore();
```

Simple enough.

And finally rendering our component

```typescript
await render(
  <Provider store={mockStore}>
    <MoviePanelMock />
  </Provider>
);
```

**Notice** that we are awaiting for the render function, if we don't do this then our test won't wait for reducer to finish (even though dispatches in redux are synchronous)

In the end our sample test can look like this.

```typescript
describe("Movie Panel", () => {
  let mockStore: Store<AppState, AppActions>;
  let MoviePanelMock: FC;

  beforeEach(() => {
    mockStore = configureStore();
    MoviePanelMock = connect(
      mapStateToProps,
      (dispatch: ThunkDispatch<AppState, undefined, AppActions>) =>
        mapDispatchToProps(getAllMoviesMock)(dispatch)
    )(MoviePanel);
  });

  afterEach(cleanup);

  test("Should clear movies after pressing clear movies button", async () => {
    // GIVEN
    await render(
      <Provider store={mockStore}>
        <MoviePanelMock />
      </Provider>
    );

    // WHEN
    fireEvent.click(screen.getByTestId("clear-movies-button"));

    // THEN
    expect(screen.queryByTestId("movie-list-component")).toBeEmpty();
  });
});
```

## How are we able to inject dependencies in `connect` function?

Let's take a look into the `connect` function. What does the connect function do?

```typescript
function connect(mapStateToProps?: function, mapDispatchToProps?: function | object, mergeProps?, options?): ConnectedComponent
```

To perform injection, we will take a closer look into `mapDispatchToProps` argument.

`connect`

```typescript
    <TStateProps = {}, TDispatchProps = {}, TOwnProps = {}, State = DefaultRootState>(
        mapStateToProps: MapStateToPropsParam<TStateProps, TOwnProps, State>,
        mapDispatchToProps: MapDispatchToPropsNonObject<TDispatchProps, TOwnProps>
    ): InferableComponentEnhancerWithProps<TStateProps & TDispatchProps, TOwnProps>;
```

Yeah... that does not look simple, we need to go deeper.

`mapDispatchToProps`

```typescript
export type MapDispatchToPropsNonObject<TDispatchProps, TOwnProps> =
  | MapDispatchToPropsFactory<TDispatchProps, TOwnProps>
  | MapDispatchToPropsFunction<TDispatchProps, TOwnProps>;
```

Not deep enough

`MapDispatchToPropsFunction`

```typescript
export type MapDispatchToPropsFunction<TDispatchProps, TOwnProps> = (
  dispatch: Dispatch<Action>,
  ownProps: TOwnProps
) => TDispatchProps;
```

Ok we can work with that.  
As you can see our mapDispatchToProps should be a function that has two parameters  
`(dispatch: Dispatch<Action>, ownProps: TOwnProps)`
and this is the information that we were looking for.

Let's create that function  
`MoviePanel.component.tsx`

```typescript
...
export const mapDispatchToProps= (dispatch: ThunkDispatch<AppState, undefined, AppActions>): LinkDispatchProps => ({
    retrieveMovies: () => dispatch(retrieveMoviesAction(getAllMoviesREST)),
    clearMovies: () => dispatch(clearMovies())
});

export default connect(mapStateToProps, mapDispatchToProps)(MoviePanel);
```

We are not using second paramter `ownProps` so we can omit it in JavaScript  
But wait what's that the `dispatch` parameter was of type `Dispatch<Action>` and not `ThunkDispatch<AppState, undefined, AppActions>` is it a mistake?

No, thanks to ReduxThunk middleware standard redux dispatch is enhanced with thunk dispatch and now it's type looks like this `ThunkDispatch<AppState, undefined, AppActions>`

Since second parameter of `connect` function must be a function that takes `dispatch: ThunkDispatch<AppState, undefined, AppActions>` we can wrap our `mapDispatchToProps` function into anonymous function and pass `dispatch` parameter to `mapDispatchToProps` explicitly.

```typescript
...
export const mapDispatchToProps = (dispatch: ThunkDispatch<AppState, undefined, AppActions>): export const mapDispatchToProps= (dispatch: ThunkDispatch<AppState, undefined, AppActions>): LinkDispatchProps => ({
    retrieveMovies: () => dispatch(retrieveMoviesAction(getAllMoviesREST)),
    clearMovies: () => dispatch(clearMovies())
});

export default connect(mapStateToProps, (dispatch) => mapDispatchToProps(dispatch))(MoviePanel);
```

Now we have full control over when to pass `dispatch` to `mapDispatchToProps` allowing us to create *Higher Order Function* and apply *Inversion of Control* to get rid of this nasty concrete implementation of `getAllMoviesREST` in our component.  
Are you still with me? Good, let's just do that.

Time to move `getAllMoviesREST` to a parameter of `mapDispatchToProps`

```typescript
...
export const mapDispatchToProps = (getAllMovies: () => Promise<MovieDTO[]>) => (dispatch: ThunkDispatch<AppState, undefined, AppActions>): LinkDispatchProps => ({
    retrieveMovies: () => dispatch(retrieveMoviesAction(getAllMovies)),
    clearMovies: () => dispatch(clearMovies())
});
```

As you can see instead of adding another parameter next to `dispatch` in `mapDispatchToProps` we are wrapping it into another function by using *currying*. Thanks to this we won't be interfering with standard interface of `mapDispatchToProps` function (which takes ownProps as second parameter) allowing us to write it in more programmer friendly syntax.

Like this:

```typescript
export const mapDispatchToProps = (getAllMovies: () => Promise<MovieDTO[]>) => (
  dispatch: ThunkDispatch<AppState, undefined, AppActions>
): LinkDispatchProps => ({
  retrieveMovies: () => dispatch(retrieveMoviesAction(getAllMovies)),
  clearMovies: () => dispatch(clearMovies())
});

export default connect(
  mapStateToProps,
  mapDispatchToProps(getAllMoviesREST)
)(MoviePanel);
```

This is also valid, but more explicit

```typescript
export const mapDispatchToProps = (getAllMovies: () => Promise<MovieDTO[]>) => (
  dispatch: ThunkDispatch<AppState, undefined, AppActions>
): LinkDispatchProps => ({
  retrieveMovies: () => dispatch(retrieveMoviesAction(getAllMovies)),
  clearMovies: () => dispatch(clearMovies())
});

export default connect(
  mapStateToProps,
  (dispatch: ThunkDispatch<AppState, undefined, AppActions>) =>
    mapDispatchToProps(getAllMoviesREST)(dispatch)
)(MoviePanel);
```

Now we can easily inject dependencies into our components allowing it to be free of slow and unreliable communication means such as API calls, which makes tests faster and easier to maintain.

Huge shout out to [Jacek Lipiec](https://github.com/venthe) for helping me to figure this stuff out!
