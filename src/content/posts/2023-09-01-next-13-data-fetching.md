---
title: Next.js 13 Data Fetching with the newest App Router
description: How to fetch data with the new Next.js 13 App Router, and some tips and tricks.
coverImage: ./2023-09-01-next-13-data-fetching.jpg
coverImageAlt: A close-up shot of a mining rig with multiple GPUs, cooling fans, and power cables, all neatly arranged on a motherboard within a metal frame.
publishDate: 2023-09-01T10:34:35+08:00
author: cpeaustriajc
keywords:
  - Web Development
  - Next.js
  - App Router
  - Astro
  - Vercel
draft: false
---

With the new release of Next.js 13 came with new paradigms and concepts. One prevalent concept that react and next.js introduced
are server components, which are components that are rendered on the server. This allows for faster page loads and better SEO.

This is a great concept where you can do asynchronous tasks on a react component such as data fetching. This means you won't need
`useEffect` to get data from an API; it is considered an anti-pattern in react when it comes to fetching data on the client with
`useEffect` because it can cause some expensive problems on the long run if you are not careful with it.

One of the problems that the react documentation explained are [race conditions][race-conditions] which are explained on the link.

It is much better for SEO and performance to fetch data on the server, and Next.js 13 makes it easier to do so. In this article
we will be going over how to fetch data with the new Next.js 13 App Router, and some tips and tricks.

## Solution 1: Fetching data with `fetch` and React Server Components

This is as simple as it gets, create an async component, fetch data with `fetch` on the top level then access the data on the
template.

```jsx
// app/page.jsx

export default async function Page() {
  const res = await fetch("https://jsonplaceholder.typicode.com/posts");
  const posts = await res.json();

  return (
    <div>
      <h1>Posts</h1>
      <ul>
        {posts.map((post) => (
          <li key={post.id}>{post.title}</li>
        ))}
      </ul>
    </div>
  );
}
```

That's it! It will fetch the data on the server and serve an html page with the data already fetched. If you want to revalidate or refetch
the data, Next.js extends the `fetch` where the framework allows you to configure the caching and revalidation behavior for each request
on the server.

In the previous example, we can further improve the fetch api by adding the `revalidate` property to the response object and doing error handling.

Let's say we want to revalidate the data every 1 hour.

```jsx
// app/page.jsx

export default async function Page() {
  const res = await fetch("https://jsonplaceholder.typicode.com/posts", {
    next: { revalidate: 3600 }, // Will revalidate every 1 hour
  });
  const posts = await res.json();

  return (
    <div>
      <h1>Posts</h1>
      <ul>
        {posts.map((post) => (
          <li key={post.id}>{post.title}</li>
        ))}
      </ul>
    </div>
  );
}
```

After that, we can add a throw statement to throw an error if the response status is not 200.

```jsx
// app/page.jsx

export default async function Page() {
  const res = await fetch("https://jsonplaceholder.typicode.com/posts", {
    next: { revalidate: 3600 }, // Will revalidate every 1 hour
  });

  if (!res.ok) {
    // This will activate the closest `error.js` Error Boundary.
    throw new Error(`Failed to fetch the data`);
  }
  const posts = await res.json();

  return (
    <div>
      <h1>Posts</h1>
      <ul>
        {posts.map((post) => (
          <li key={post.id}>{post.title}</li>
        ))}
      </ul>
    </div>
  );
}
```

But what if you really need to fetch data on the client? This is where the second solution comes in.

## Solution 2: Fetching data in the client with third party libraries

Like I said, it is better to fetch data on the server for SEO and performance because the data that will be fetched will also be included in the HTML.

But if client side data fetching is needed, consider using a third party library first like SWR and React Query to avoid using `useEffect`

This is because of the edge cases that these library have are already included like deduplicating requests, caching responses,
avoiding network waterfalls, refetching, mutation, etc.

Let's take a look on what it looks like when you want to fetch
data with SWR

SWR is a data fetching library created by vercel which is a much
more simpler solution compared to react query.

Let's say we want to fetch data in a client component.

```jsx
// components/profile.tsx
import { getProfileById } from '@/lib/actions'; // Example import

export function Profile({id}: {id: string}) {
     const { data, error, isLoading } = useSWR([`/api/profile/`, id], ([url, userId]) => getProfileById(url, userId))

    if (error) return <div>failed to load</div>

    if (isLoading) return <div>Loading...</div>

    return <div>hello, {data?.name}</name>
}
```

It looks similar, but the difference is that it is fetching directly in the browser. If you look at the difference between the network requests in your devtools you would see the request
being made.

### Improving Data Fetching with Suspense

This can be further improved by using the Suspense component. Suspense in react is a component that replaces a fallback component until the children that is wrapped around it has finished loading.

We can do this by the following example:

```jsx
// components/profile.tsx
'use client'

import { getProfileById } from '@/lib/actions'; // Example import


export function Profile({ id }: { id: string }) {
    const { data, error, isLoading } = useSWR([`/api/profile/`, id], ([url, userId]) => getProfileById(url, userId), { suspense: true })

    return <div>hello, {data.name}</name>
}
```

then on, `app/page.tsx`:

```jsx
// app/page.tsx
"use client";

import { Suspense } from "react";
import { Skeleton } from "@/components/ui/skeleton"; // Example
import { Profile } from "@/components/profile";
import { ErrorBoundary } from "react-error-boundary"; // npm install react-error-boundary

export default function Page() {
  return (
    <ErrorBoundary fallback={<h2>Could not fetch profile</h2>}>
      <Suspense fallback={<Skeleton />}>
        <Profile />
      </Suspense>
    </ErrorBoundary>
  );
}
```

It looks much cleaner and doesn't actually need to use optional chaining operator and conditional checks.

**IMPORTANT NOTE**

> React Suspense is still in experimental when it comes to data
> fetching but the react team is working on how to properly
> handle suspense when it comes to data fetching.

> React Suspense **does not** detect when data is fetched inside > an Effect or event handler.

## Tips and Tricks

Here are some good tips and tricks when it comes to data fetching in react.

1. Avoid data fetching in the client as much as possible.
2. Avoid using `useEffect` for data fetching, use an opinionated data fetching library to use the full potential of what react can offer when handling async data.
3. In my opinion it is considered bad practice to add `'use client'` in page.tsx or layout.tsx because ideally these should be rendered on the server.
4. When dealing with search fields or input data, it is good to consider using debounce or throttling techniques to reduce the amount of requests and to avoid api abuse.

## Conclusion

In this article, we learned how we can fetch data in the new
Next.js App Router in the server and the client. We also had
some quick discussion react suspense and how we can use it to
display loading data.

I hope you learned something on this article and apply it on
your current and new projects.

### Follow me on social media!

I am active on the following social media platforms.

- [Twitter](https://twitter.com/cjfujiwara)
- [GitHub](https://github.com/cpeaustriajc)

### References

- [Race Conditions in React][race-conditions]
- [useEffect Hook Reference][use-effect]
- [Suspense Component API reference][suspense]
- [Data Fetching Documentation Next.js][data-fetching-next]
- [SWR Library][swr]

[race-conditions]: https://maxrozen.com/race-conditions-fetching-data-react-with-useeffect
[suspense]: https:/react.dev/reference/react/suspense
[data-fetching-next]: https://nextjs.org/docs/app/building-your-application/data-fetching/fetching-caching-and-revalidating
[swr]: https://swr.vercel.app
[use-effect]: https://react.dev/reference/react/useEffect
