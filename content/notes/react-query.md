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

**Marking cache as stale and triggering a re-fetch**
- `staleTime` - timer of how long to cache query data, defaults to `0`, set to `Infinity` to never expire
- `cacheTime` - how long data will remain in memory, defaults to 5 minutes -- assuming after 5 minutes reverts to `loading` etc
- Triggers
    - Query is fetch when component mounts (isLoading)
    - On window focus, stale queries are re-fetched (i.e. when switching back to browser tab, etc). `refetchONWindowFocus` defaults to `true`
    - On network re-connect - if network connection is lost, will re-fetch, `refetchOnReconnect` defaults to `true`
    - Interval - `refetchInterval` - re-fetches even if cache is still `fresh` - use case for rapidly changing data (stock ticker or messages?)
- Invalidate cache after mutation
