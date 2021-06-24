[React-table](https://react-table.tanstack.com/) and [react-query](https://react-query.tanstack.com/) are two awesome react libraries from [Tanner](https://github.com/tannerlinsley). They are well documented [here](https://tanstack.com/) with the examples of most use cases. Now, I am going to explain how you can leverage react-table and react-query for server side pagination.

[React-table](https://react-table.tanstack.com/) is a powerful headless design so you can have full control over the render and style aspects. It gives you all the declarative hooks APIs for you to compose and conquer. In order to control the pagination, we need to use `usePagination` with `manualPagination: true`. 

[React-query](https://react-query.tanstack.com/) is a declarative and automatic server state library to fetch and cache data from your backend APIs. For our purpose, `useQuery` with `keepPreviousData` option will enable **the data from the last successful fetch available while new data is being requested, even though the query key has changed** ([For more info](https://react-query.tanstack.com/guides/paginated-queries#better-paginated-queries-with-keeppreviousdata)). 

To explain further, let's consider an example of building a server side paginated table using the [Pokémon](https://pokeapi.co/) API.

```javascript
const fetchPokemonData = async (page, pageSize) => {
  const offset = page * pageSize;
  try {
    const response = await fetch(
      `https://pokeapi.co/api/v2/pokemon?offset=${offset}&limit=${pageSize}`
    );
    const data = await response.json();

    return data;
  } catch (e) {
    throw new Error(`API error:${e?.message}`);
  }
};
```
Since Pokémon API expects offset, it's being derived from page and pageSize.


```javascript
const { isLoading, error, data, isSuccess } = useQuery(
  ['players', queryPageIndex, queryPageSize],
  () => fetchPokemonData(queryPageIndex, queryPageSize),
  {
    keepPreviousData: true,
    staleTime: Infinity,
  }
);
```
This fetches the data as and when the query keys, which are the page and pageSize from the state change. `staleTime` is marked as infinity as we don't want to burden the Pokémon API with too many hits.

Now, let's bring in `useTable` hook from [react-table](https://react-table.tanstack.com/).

```javascript
const {
  getTableProps,
  getTableBodyProps,
  headerGroups,
  prepareRow,
  page,
  canPreviousPage,
  canNextPage,
  pageOptions,
  pageCount,
  gotoPage,
  nextPage,
  previousPage,
  setPageSize,
  // Get the state from the instance
  state: { pageIndex, pageSize },
} = useTable(
  {
    columns,
    data: isSuccess ? trimData(data.results) : [],
    initialState: {
      pageIndex: queryPageIndex,
      pageSize: queryPageSize,
    },
    manualPagination: true, // Tell the usePagination
    // hook that we'll handle our own data fetching
    // This means we'll also have to provide our own
    // pageCount.
    pageCount: isSuccess ? Math.ceil(totalCount / queryPageSize) : null,
  },
  usePagination
);
```
We are passing the `queryPageIndex` and `queryPageSize` as the initial state. When the fetch query is `isSuccess`, we pass on the `data` and the `pageCount`. Let's look at how we can keep our own state.

```javascript
const initialState = {
  queryPageIndex: 0,
  queryPageSize: 10,
  totalCount: null,
};

const PAGE_CHANGED = 'PAGE_CHANGED';
const PAGE_SIZE_CHANGED = 'PAGE_SIZE_CHANGED';
const TOTAL_COUNT_CHANGED = 'TOTAL_COUNT_CHANGED';

const reducer = (state, { type, payload }) => {
  switch (type) {
    case PAGE_CHANGED:
      return {
        ...state,
        queryPageIndex: payload,
      };
    case PAGE_SIZE_CHANGED:
      return {
        ...state,
        queryPageSize: payload,
      };
    case TOTAL_COUNT_CHANGED:
      return {
        ...state,
        totalCount: payload,
      };
    default:
      throw new Error(`Unhandled action type: ${type}`);
  }
};

const [{ queryPageIndex, queryPageSize, totalCount }, dispatch] =
  React.useReducer(reducer, initialState);
```
I am using `useReducer` in this case. As `queryPageIndex` and `queryPageSize` are being used in the `useQuery` keys, the `fetchPokemonData` is invoked when we either move to a new page or change to a new pageSize. Since we are using `staleTime: Infinity`, already visited pages with a particular page size is served from the cache for infinite time.

```javascript
React.useEffect(() => {
  dispatch({ type: PAGE_CHANGED, payload: pageIndex });
}, [pageIndex]);

React.useEffect(() => {
  dispatch({ type: PAGE_SIZE_CHANGED, payload: pageSize });
  gotoPage(0);
}, [pageSize, gotoPage]);

React.useEffect(() => {
  if (data?.count) {
    dispatch({
      type: TOTAL_COUNT_CHANGED,
      payload: data.count,
    });
  }
}, [data?.count]);
```

Here comes the interesting part where we capture `pageIndex` and `pageSize` of react-table state changes in the useEffect and dispatch to keep a copy in our local state. This is clearly duplicating them in favour of using `useQuery` in its declarative nature. There is another option to impertatively use react-query's `fetchQuery` and keep the data in the local state but you will miss the status and all other automagical stuff of `useQuery`. If you want to explore more on this topic, you can follow the reference links given at the bottom.



*References*:

https://github.com/tannerlinsley/react-query/discussions/736#discussioncomment-227931

https://github.com/tannerlinsley/react-table/discussions/2193

https://github.com/tannerlinsley/react-query/discussions/1113


Photo by <a href="https://unsplash.com/@jwwhitt?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Jordan Whitt</a> on <a href="https://unsplash.com/@jwwhitt?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>