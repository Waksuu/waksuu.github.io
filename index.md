


# Black box Integration Testing React components connected to Redux

Whats the better way to make your code more maintainable and easier to change in the future than writing tests?
Probably hiring a bunch of interns to test code manually after each commit, but we don't have time and money for that so tests will have to do.

In this post I will focus on the importance of black box testing and how this can be achieved  in React.
I will be using jest, react-testing-liblary, redux-thunk and Typescript for my own sanity.</br>
I am also gonna assume that you know the basics of these technologies.

In addition I will only focus on Integration Testing (how to test flow of your component without need for backend) because I don't feel that there are good resources for that and Unit Testing is well examined.

## Black box testing, why bother?

I am big fan of black box testing, especially when working with front-end. In the world of <i>javascript frameworks chaning so fast that before you download npm dependencies this dependency is already outdated</i> I cannot imagine writing tests that can be easliy broken by the change in implementation that doesn't not have any impact on the behaviour of the application. </br>
This kind of enviroment discourages developers from refactoring and writing tests because everything you touch has high change of breaking something (be it tests or the actual code).

We can avoid this problemy by changing the way of thinking about our code and trying to test <i> behaviour </i> and not the internal implementation (that can be easlily changed). Doing so greatly reduces the need of changing tests when we want to change the way that our code works (e.g. refactoring code for optimization reasons).

For example, when testing let's say button counter component, I shouldn't care if the way of storing button counter is implemented within the component as internal state or Redux deals with it, this is an <i>implementation detail</i> that can change, but the <i> behaviour </i> of the button will stay the same.

You might say, ok this is a very basic exmaple, real world is not that easy and components have some kind of dependencies (e.g component is calling some endpoint to retrieve data), and how do I test it?

## Test scenario

Let's say that we want to test component that is responsible for managing movies, and for this basic example our component will have two methods: ``retrieveMovies()`` and ``clearMovies()``.

 `MoviePanel.component.tsx`
```` typescript
type Props = LinkStateProps & LinkDispatchProps;

const MoviePanel: FC<Props> = (props: Props) => {
    useEffect(() => {
        props.retrieveMovies();
    }, [])

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

const mapDispatchToProps = (dispatch: ThunkDispatch<AppState, any, AppActions>): LinkDispatchProps => ({
    retrieveMovies: () => dispatch(retrieveMoviesAction()),
    clearMovies: () => dispatch(clearMovies())
});

export default connect(mapStateToProps, mapDispatchToProps)(MoviePanel);
````

 `Movie.action.ts`
```` typescript
export const retrieveMoviesAction = (): ThunkAction<Promise<void>, AppState, undefined, AppActions> => {
    return async (dispatch: ThunkDispatch<AppState, any, AppActions>) => {
        const movies: MovieDTO[] = await getAllMoviesREST();
        dispatch(moviesRetrieved(movies));
    };
};

const moviesRetrieved = (movies: MovieDTO[]): RetrieveMovieListAction => {
    return {
        type: MovieListRetrieved,
        movies
    };
}

export const clearMovies = (): ClearMovieListAction => {
    return {
        type: ClearMovieListRequest,
    };
}

````

 `Movie.reducer.ts`
```` typescript
const moviesInitialState: MovieDTO[] = [];

export const movieReducer = (state = moviesInitialState, action: MovieListActionTypes): MovieDTO[] => {
    switch (action.type) {
        case MovieListRetrieved:
            return action.movies;
        case ClearMovieListRequest:
            return [];
        default:
            return state;
    }
};
````

Let's assume that method `getAllMoviesREST()`in  `Movie.action.ts` is an API call that returns promise  (for the simplicity of this example under the hood it is just a mock but I'll leave it to your imagination to do the REST).
Now there is one problem, how do we write test for a component that is depended on external API?
Well there are two most popular options:

You can intercept all API calls with some test interceptor (external liblary) but that will leave your test fragile and hard to mock up (any change in the API method will force you to do some changes in test) and we want to avoid that. </br>
Or we can apply <i>Inversion of Control</i> principle and take our dependency (the `getAllMoviesREST()` method) as a parameter to action (as a method reference) and then compose our component, with all of its dependecies in a  `MoviePanel.component.tsx`.

## Making our component testable
Firstly let's make our action method (`retrieveMoviesAction()`in  `Movie.action.ts`) independent of concrete implementation of `getAllMoviesREST()` by simply recieving this method as function parameter.

 `Movie.action.ts`
````typescript 
export const retrieveMoviesAction = (getAllMovies: () => Promise<MovieDTO[]>): ThunkAction<Promise<void>, AppState, undefined, AppActions> => {
    return async (dispatch: ThunkDispatch<AppState, any, AppActions>) => {
        const movies: MovieDTO[] = await getAllMovies();
        dispatch(moviesRetrieved(movies));
    };
};
````
Simple enough.

Ok but now our  `MoviePanel.component.tsx` is complaining that function `retrieveMoviesAction()` requires 1 argumnet and not 0.

Now you might be temptend to just directly add conrete implementation of our `getAllMovies` method in `mapDispatchToProps` like this.

 `MoviePanel.component.tsx`
````typescript 
const mapDispatchToProps = (dispatch: ThunkDispatch<AppState, any, AppActions>): LinkDispatchProps => ({
    retrieveMovies: () => dispatch(retrieveMoviesAction(getAllMoviesREST)),
    clearMovies: () => dispatch(clearMovies())
});
````

But this solution still blocks us from injecting mocks into our component. </br>
Now we need to apply <i>Inversion of Control</i> principle for `mapDispatchToProps` in conjuction with <a href="https://blog.bitsrc.io/understanding-currying-in-javascript-ceb2188c339" target="_blank">currying</a>.

````typescript 
const mapDispatchToProps = (
        getAllMovies: () => Promise<MovieDTO[]>
    ) => (dispatch: ThunkDispatch<AppState, any, AppActions>): LinkDispatchProps => ({
    retrieveMovies: () => dispatch(retrieveMoviesAction(getAllMovies)),
    clearMovies: () => dispatch(clearMovies())
});

export default connect(mapStateToProps, (
        dispatch: ThunkDispatch<AppState, any, AppActions>
    ) => mapDispatchToProps(getAllMoviesREST)(dispatch))
(MoviePanel);
````

Notice that we pass `getAllMoviesREST` in the connect function, allowing connect function to compose our component.

I will provide detailed explanation on how it exactly works at the end of this post.

In order to create MoviePanel component in tests we have to add few exports </br>
 `MoviePanel.component.tsx`
````typescript 
export const MoviePanel: FC<Props> = (props: Props) => {
	...
};
````
````typescript 
export  const  mapStateToProps = (state: AppState): LinkStateProps  => ({
	...
});
````
````typescript 
export const mapDispatchToProps = (
	...
});
````
And in order to check if our component was changed upon some action we need to add ``data-testid`` in two places. </br>
``MovieControls.component.tsx``
````typescript 
export const MovieControls: FC<Props> = (props: Props) => {
    return (
        <button data-testid="clear-movies-button" onClick={props.clearMovies} className="my-button">
            Clear movies!
        </button>
    );
};
````
``MovieList.component.tsx``
````typescript
export const MovieList: FC<Props> = (props: Props) => {
    return (
        <div data-testid="movie-list-component">
            {props.movies.map(movie =>
                <div key={movie.id}>{movie.name}</div>
            )} 
        </div>
    );
};
````
## Testing our component
Testing components connected to redux is very simmilar to testing components typical components. The main difference is that we have to create component using ``connect`` function, wrap our component in ``Provider`` component and create mock store. 

Let's start with creating our component with ``connect`` function. </br>

````typescript 
import { mapStateToProps, mapDispatchToProps, MoviePanel } from  "../../../app/movie/MoviePanel.component";
...
const MoviePanelMock: FC = connect(mapStateToProps, (
            dispatch: ThunkDispatch<AppState, any, AppActions>
        ) => mapDispatchToProps(getAllMoviesMock)(dispatch))(MoviePanel);
        
const getAllMoviesMock = (): Promise<MovieDTO[]> => {
    return Promise.resolve(mockMovies);
}

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
````
And that's it, now we created mock component with injected dependencies. 

Now we need to create store

````typescript 
import { configureStore, AppState } from  "../../../common/redux/store/ConfigureStore";
...
const mockStore: Store<AppState, AppActions> = configureStore();
````
Simple enough.

And finally rendering our component
````typescript 
await render(
	<Provider store={mockStore}>
		<MoviePanelMock />
	</Provider>
)
````
<b>Notice</b> that we are awaiting for the render function, if we don't do this then our test won't wait for reducer to finish (even though dispatches in redux are synchronous)

In the end our sample test can look like this.

````typescript 
describe("Movie Panel", () => {
    let mockStore: Store<AppState, AppActions>;
    let MoviePanelMock: FC

    beforeEach(() => {
        mockStore = configureStore();
        MoviePanelMock = connect(mapStateToProps, (
            dispatch: ThunkDispatch<AppState, any, AppActions>
        ) => mapDispatchToProps(getAllMoviesMock)(dispatch))(MoviePanel);
    });

    afterEach(cleanup);
    
    test("Should clear movies after pressing clear movies button", async () => {
        // GIVEN
        await render(
            <Provider store={mockStore}>
                <MoviePanelMock />
            </Provider>
        )

        // WHEN
        fireEvent.click(screen.getByTestId("clear-movies-button"));

        // THEN
        expect(screen.queryByTestId("movie-list-component")).toBeEmpty();
    })
})
````

All code samples can be found on my <a href="https://github.com/Waksuu/react-exploratory/tree/after-redux-tests">github</a>
