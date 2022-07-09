---
title: "React Query"
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


