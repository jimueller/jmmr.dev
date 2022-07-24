---
title: "react-query"
date: 2022-07-09T14:09:28-05:00
categories:
 - javascript 
tags:
 - react
slug: react-query
---

# React Query Notes

## Query Key

Match query key to the rest api in the order of least to most specific.  Name the query, then path params, and finally put query params in an object.

```javascript

// Given this url
// `https://api.github.com/repos/:owner/:repo/issues?state=open`


// Match least to most specifc in query key
const issueQuery = useQuery(["issues", owner, repo, { state:"open"}], queryFunc);
```
## Create Custom Hook

It's possible to use the useQuery hook directly in a component, but it's better to extract it into it's own hook. This allows re-use across multiple components and protects against
accidental re-use of query keys across separate queries.

## Parallel, Dependent, and Deferred Queries

### Parallel Queries

Returning multiple requests in one query function.  There are a couple of options, using plain old `Promise.all()` or using the `useQueries()` hook.

#### Separate useQuery functions

Can just have two separate `useQuery` functions.  Render may look odd basically two renders will happen and may be missing data.

#### Promise.all()

Create a query function to load multiple requests, most simplistic but has limited options on error handling and performance

```js
function getBranchesAndTags(repoId) {
  return Promise.all([
   fetch(`https://api.github.com/repos/${repoId}/branches`).then(res => res.json()),
   fetch(`https://api.github.com/repos/${repoId}/tags`).then(res => res.json()),
  ])
}

const branchsAndTagsQuery = useQuery(["branchesAndTags", repoId], () => getBranchesAndTags(repoId));

// states are now combined, both must succeed / load
if(branchesAndTagsQuery.isLoading){
   return <div>Loading...</div>
}
```

**Considerations**

- Do you need a separate state management (loading, error handling)?  Would you want to still display one if the other fails?
- Would it be better to have a separate cache?  If query could be re-used elsewhere it may save an api call to the server

#### useQueries

Basically the same as above, but with a hook.

```js
const results = useQueries({
  queries: [
    { queryKey: ['post', 1], queryFn: fetchPost },
    { queryKey: ['post', 2], queryFn: fetchPost }
  ]
})
```


Reference: [useQueries](https://tanstack.com/query/v4/docs/reference/useQueries)

### Dependent Queries

Use async/await and the `enabled: true|false` configuration option.  You can get crazy with with the `fetchStatus` to handle when all data is ready to render.

### Deferred Queries

Similar to dependent queries, but wait for user input.  Basically use the same `enabled` flag to only process when there is input.  Would need to debounce on a "typeahead" style search or only search on `Enter` or button click.  Take advantage of the query key / cache for duplicated search inputs. 

## States

### Main states

- `loading` - first time it loads when there is no cached data or when query has been invalidated -- see `fetchStatus: "fetching"` for subsequent loads
- `error` - query function threw an error
- `success` - no error from query function

### Fetch Status

- `idle` - doesn't need to be fetched (cache fresh?)
- `fetching` - currently being fetched
- `paused` - tried to fetch, but was prevented.  Possible reasons could be network offline or query is disabled (need to confirm this).

### Cache status

- `stale` - query will be refetched from the server
- `fresh` - cache data will be used

#### Marking cache as stale and triggering a re-fetch

- `staleTime` - timer of how long to cache query data, defaults to `0`, set to `Infinity` to never expire
- `cacheTime` - how long data will remain in memory, defaults to 5 minutes -- assuming after 5 minutes reverts to `loading` etc

##### Automatic triggers

- Query is fetch when component mounts (isLoading)
- On window focus, stale queries are re-fetched (i.e. when switching back to browser tab, etc). `refetchOnWindowFocus` defaults to `true`
- On network re-connect - if network connection is lost, will re-fetch, `refetchOnReconnect` defaults to `true`
- Interval - `refetchInterval` - re-fetches even if cache is still `fresh` - use case for rapidly changing data (stock ticker or messages?)

##### Manual Triggers

Use cases

- Invalidate cache after mutation
- Web socket message -- chat to get latest messages / notifications, real time update notifications
- Refresh button

Typically invalide queries and let react-query decide what to re-fetch based on query key **and** if the query is active.  Will smartly not re-fetch queries that are not active on the page.

You can call `refetchQueries` if you know you need to force a re-fetch

By specifiying a query key to invalidate or refetch, react-query will determine which queryies need to be refetched.

```javascript
// Note: you wouldn't actually write separate todos queries like this
const userQuery = useQuery(['users']);                            // 1
const allTodosQuery = useQuery(['todos']);                        // 2   
const usersTodosQuery = useQuery(['todos', userId]);              // 3
const usersOpenTodosQuery = useQuery(['todos', userId, 'open']);  // 4

queryClient.refetchQueries(['users']); // => refetches 1
queryClient.refetchQueries(['todos']); // => refetches 2,3,4
queryClient.refetchQueries(['todos', userId]); // => refetches 3,4
queryClient.refetchQueries(['todos', userId, 'open', {exact:true}); // => only refetches #4
```

There are other options to match certain querys such as `type: 'active'|'stale'|'all'`

## Error Handling

- Query function must throw an error or reject the promise 
- `fetch` only rejects on 5xx, axios is cool though (and has a config option for which status codes throw)

### Retries

Retries only happen when re-fetches, if initial fetch fails, query goes into `status='error'` immediately.  Defaults to 3 retries with exponential backoff, should be good for most use cases, can be overridden with `retry` and `retryDelay` configs.

### ErrorBoundary

Use react's error boundaries to make handling errors much easier.  Supported by react-query with the `useErrorBoundary:true` config.  Check out `react-error-boundary` package.

### onError callback

Do whatever you need to in the callback, such as logging, tracing, or showing an ephemeral error message.

### Using cached data

Cached data is not updated or cleared on an error, so if your use case permits, you can show the cached data along with an error message.  This is not compatible with `ErrorBoundary`

## Preloading Data

There are two main ways of preloading data, priming the query cache with data already loaded or by prefetching.

### Cache Priming

If you have a query that gets a collection of items, and another query that gets a single item by id, you can prime the "by id" cache with the results of the collection query.

```javascript
// queryClient passed in from component useQueryClient
async function fetchCars(queryClient){
  const cars = await fetch(`/api/cars/${make}/`).then(res=>res.json());

  // prime individual car query using the query key
  cars.forEach(car => {
    // puts car data in cache, !! imporant that query key matches
    queryClient.setQueryData(['cars', car.model], car);
  }

  return cars;
}
```

You could do somthing similar with `initalData` option on `useQuery()`

```javascript
const car = {}; // <= some car object that was already loaded

const carQuery = useQuery(['cars', model], ()=>fetchCar(model), {
  initialData: car
});
```

### Prefetching

If you have a good idea where the user might go next, you can prefetch data.  An example is when hovering over a link, you can prefech the data for the next "page".

```javascript

export default function CarList(){
  const queryClient = useQueryClient();
  const cars = useQuery();

  return (
    <ul>
      cars.map(car => {
       return <li key={car.model} onMouseOver={()=>{
	      // load the data that will be needed on the car model page if link is clicked
          queryClient.prefechQuery(['cars', car.model], ()=> fetchCar(car.model));
	   }>{car.model}</li>
	  }
	</ul>
  )
}

```

### Placeholder

A final option is to load some placeholder "fake" or static data that will be replaced with the actual query data. This data will _not_ be retained in the query cache.

```javascript
const makes = ["Ford", "Chevy", "Ferrari"]; // <= some static data that isn't likely to change

const makesQuery = useQuery(['makes'], ()=> fetchMakes(), {
  placeHolderData: makes
});

```

### Considerations

If you are using a stale time with your queries, you can tell react-query what the "load" time of the `initialData` is, so it will know whether to re-fetch. Refer to [`initialDataUpdatedAt`](https://tanstack.com/query/v4/docs/guides/initial-query-data#staletime-and-initialdataupdatedat)
