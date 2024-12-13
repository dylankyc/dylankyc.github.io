# Setup Tanstack React Query in NextJS Project

<!-- toc -->

# Introduction

In the modern web development landscape, efficiently managing and handling data is a crucial aspect that can dramatically impact the user experience. React Query is a powerful data synchronization and state management library that aims to simplify fetching, caching, background updates and server state management in your React applications.

Next.js is a popular React framework that enables features such as Server-Side Rendering and Static Site Generation out of the box. This makes it a great choice for building performant, SEO-friendly web applications. However, when it comes to data fetching and managing server state, Next.js leaves the choice up to the developer.

Therefore, using React Query with Next.js can be a powerful combination for building robust, data-driven applications with improved user experience. This blog post will guide you step by step to integrating React Query in a Next.js project.

We will cover:

- Setting up a new Next.js project
- Installing and setting up React Query
- Fetching data using React Query
- Displaying the fetched data using a table component

No matter the size and scale of your application, using React Query with Next.js can improve the efficiency of your data operations and the overall quality of your product. Let's dive in!

# Install

To install tanstack/react-query, you can use yarn or npm:

```bash
yarn add @tanstack/react-query
yarn add -D @tanstack/eslint-plugin-query
```

This will install tanstack/react-query (`"@tanstack/react-query": "^5.21.7"`) as a dependency in your project. See package.json for more details.

```json
{
  "name": "mui-5-example",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint"
  },
  "dependencies": {
    "@emotion/react": "^11.11.1",
    "@emotion/styled": "^11.11.0",
    "@mui/base": "^5.0.0-beta.16",
    "@mui/icons-material": "^5.11.11",
    "@mui/material": "^5.14.9",
    "@mui/styles": "^5.11.13",
    "@mui/system": "^5.14.9",
    "@mui/utils": "^5.14.9",
    "@mui/x-data-grid": "^6.14.0",
    "@mui/x-data-grid-generator": "^6.14.0",
    "@mui/x-data-grid-pro": "^6.14.0",
    "@tanstack/react-query": "^5.21.7",
    "@types/node": "18.15.11",
    "@types/react": "18.0.31",
    "@types/react-dom": "18.0.11",
    "@visx/group": "^3.3.0",
    "@visx/mock-data": "^3.3.0",
    "@visx/responsive": "^3.3.0",
    "@visx/scale": "^3.5.0",
    "@visx/shape": "^3.5.0",
    "@visx/text": "^3.3.0",
    "@visx/wordcloud": "^3.3.0",
    "eslint": "8.37.0",
    "eslint-config-next": "13.2.4",
    "expression-eval": "^5.0.1",
    "moment-timezone": "^0.5.44",
    "next": "13.2.4",
    "qs": "^6.11.2",
    "react": "18.2.0",
    "react-dom": "18.2.0",
    "react-query": "^3.39.3",
    "recharts": "^2.5.0",
    "typescript": "5.0.3"
  },
  "devDependencies": {
    "@tanstack/eslint-plugin-query": "^5.20.1"
  }
}
```

# Use tanstack/react-query in nextjs project

After installation, you can use it in nextjs project by adding the following code in `_app.tsx`:

```typescript
import React from "react";
import "@/styles/globals.css";
import type { AppProps } from "next/app";

// import { QueryClient, QueryClientProvider } from "react-query";

import {
  Hydrate,
  QueryClient,
  QueryClientProvider,
} from "@tanstack/react-query";

// NOTE: Default code
// export default function App({ Component, pageProps }: AppProps) {
//   return <Component {...pageProps} />
// }

// NOTE: Using react-query v3, for v4+, use tanstack/react-query
// export default function App({ Component, pageProps }: AppProps) {
//   // Create a QueryClient instance
//   const queryClient = new QueryClient();

//   return (
//     // Use QueryClientProvider to wrapper your application，and pass the queryClient to the component
//     <QueryClientProvider client={queryClient}>
//       <Component {...pageProps} />
//     </QueryClientProvider>
//   );
// }

export default function App({ Component, pageProps }: AppProps) {
  // Create a QueryClient instance
  const [queryClient] = React.useState(() => new QueryClient());

  return (
    <QueryClientProvider client={queryClient}>
      {/* <Hydrate state={pageProps.dehydratedState}> */}
      <Component {...pageProps} />
      {/* </Hydrate> */}
    </QueryClientProvider>
  );
}
```

# Write your component

Now, we can write our component to use the `useQuery` hook.

The `useQuery` hook is a higher-order component that returns a `QueryClient` instance.

```typescript
import React from "react";
// import { useQuery } from "react-query";
import { useQuery } from "@tanstack/react-query";

import type { InferGetServerSidePropsType, GetServerSideProps } from "next";
import JSONTable from "../../components/table/JSONTable";

type Repo = {
  name: string;
  stargazers_count: number;
};

// Function to fetch data
const fetchData = async () => {
  const response = await fetch("https://api.github.com/repos/vercel/next.js");
  if (!response.ok) {
    throw new Error("Network response was not ok");
  }
  return response.json();
};

function DataFetchUsingTanstackReactQuery() {
  // Using the useQuery hook to fetch data
  // const { data, error, isLoading } = useQuery("fetchData", fetchData);
  const { data, error, isLoading } = useQuery({
    queryKey: ["fetchData"],
    queryFn: fetchData,
  });

  if (isLoading) {
    return <p>Loading...</p>;
  }

  if (error) {
    return <p>Error: {error.message}</p>;
  }

  return (
    <div>
      <h1>Data from API:</h1>
      <JSONTable data={data} />
    </div>
  );
}

export default DataFetchUsingTanstackReactQuery;
```

Starting with version 4 of @tanstack/react-query, there was a breaking change in how queries and mutations are called. Before version 4, it was possible to use the "array" syntax or the "object" syntax when calling query functions. After version 4, only the "object" form is allowed.

```typescript
// Older version:
// const { data, error, isLoading } = useQuery("fetchData", fetchData);
// New version:
const { data, error, isLoading } = useQuery({
  queryKey: ["fetchData"],
  queryFn: fetchData,
});
```

Below is the difference between older and newer versions of @tanstack/react-query.

```diff
- useQuery(key, fn, options)
+ useQuery({ queryKey, queryFn, ...options })

- useInfiniteQuery(key, fn, options)
+ useInfiniteQuery({ queryKey, queryFn, ...options })

- useMutation(fn, options)
+ useMutation({ mutationFn, ...options })

- useIsFetching(key, filters)
+ useIsFetching({ queryKey, ...filters })

- useIsMutating(key, filters)
+ useIsMutating({ mutationKey, ...filters })
```

# JSONTable - A component to render data as a table

As we use MUI to render the table, we write a `JSONTable` component , which takes a `data` prop and renders the data as a table.

The code is pretty simple, but we need to import the `Table` and `TableHead` components from `@mui/material`.

```typescript
import React from "react";
import {
  Table,
  TableHead,
  TableBody,
  TableRow,
  TableCell,
  TableContainer,
  Paper,
} from "@mui/material";

const JSONTable = ({ data }) => {
  const renderTableRows = () => {
    return Object.entries(data).map(([key, value]) => (
      <TableRow key={key}>
        <TableCell sx={{ wordBreak: "break-word", maxWidth: 20 }}>
          {key}
        </TableCell>
        <TableCell sx={{ wordBreak: "break-word", maxWidth: 800 }}>
          {/* // NOTE: when value is boolean, we should call toString to change it
          to string instead of JSON.stringify */}
          {typeof value === "object"
            ? JSON.stringify(value)
            : typeof value === "boolean"
            ? value.toString()
            : value}
          {/* {typeof value === "object" ? JSON.stringify(value) : value} */}
          {/* {JSON.stringify(value)} */}
          {/* {typeof value} */}
        </TableCell>
      </TableRow>
    ));
  };

  return (
    <TableContainer
      component={Paper}
      style={{ margin: "20px", marginRight: "40px" }}
    >
      <Table sx={{ minWidth: 650 }} aria-label="simple table">
        <TableHead>
          <TableRow>
            <TableCell>Property</TableCell>
            <TableCell>Value</TableCell>
          </TableRow>
        </TableHead>
        <TableBody>{renderTableRows()}</TableBody>
      </Table>
    </TableContainer>
  );
};

export default JSONTable;
```
