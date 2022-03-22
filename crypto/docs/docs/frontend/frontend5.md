# Rethinking Data Fetching

React breaks complex interfaces into reusable components, allowing developers to reason about discrete units of an application in isolation, and reducing the coupling between disparate parts of an application.

In general, the overwhelming majority of products want one specific behavior: fetch all the data for a view hierarchy while displaying a loading indicator, and then render the entire view once the data is available.

One solution (let's call it "top-down") is to have a root component declare and fetch the data required by it and all of its children. However, this would introduce coupling: any change to a child component would require changing any root component that might render it! Soon it becomes very difficult to mantain, because it requires to keep track of the fields needed and properly remove that not used anymore.

Another logical approach is to have each component declare and fetch the data it requires. This sounds great. However, the problem is that a component may render different children based on the data it received. So, nested components will be unable to render and begin fetching their data until parent components' queries have completed. In other words, _this forces data fetching to proceed in stages_ (let's call it "waterfall"). Rendering would require multiple slow, serial roundtrips.

Relay allows components to specify what data they require, but to coalesce those requirements into a single query that fetches the data for an entire subtree of components. In other words, it determines statically (i.e. before your application runs; at the time you write your code) the requirements for an entire view!

## Specifying the data requirements of a component

With Relay, the data requirements for a component are specified with [fragments](https://relay.dev/docs/guided-tour/rendering/fragments/). Fragments are named snippets of GraphQL that specify which fields to select from an object of a particular type. Fragments are written within GraphQL literals. This data is then read out from the store by calling the `useFragment` hook in a functional React component.
The second parameter (`fragmentRef`) is a fragment reference are obtained by spreading a fragment into another fragment or query.

Fragments cannot be fetched by themselves; instead, they must ultimately be included in a parent query. The Relay compiler will then ensure that the data dependencies declared in such fragments are fetched as part of that parent query.

```jsx title="/scenes/dashboard/DashboardTickerItem.js"
export default memo(function DashboardTickerItem({fragmentRef}) {
  const asset = useFragment(
    graphql`
      fragment DashboardTickerItemFragment_asset on Asset {
        symbol
        color
        price {
          currency
          lastPrice
          change24Hour
        }
      }
    `,
    fragmentRef,
  );

  // ...
});
```

:::note

Relay uses `Fragments` to declare data requirements for components, and compose data requirements together.

:::

## Lazily Fetching Queries

To fetch a query lazily, you can use the `useLazyLoadQuery` hook:

```jsx title="/scenes/dashboard/DashboardTicker.js"
import {memo} from 'react';
import {graphql, useLazyLoadQuery} from 'react-relay';

import {Ticker} from '@/components';

import DashboardTickerItem from './DashboardTickerItem';

export default memo(function DashboardTicker() {
  const data = useLazyLoadQuery(
    graphql`
      query DashboardTickerQuery {
        assets(first: 10, order: {price: {tradableMarketCapRank: ASC}}) {
          nodes {
            symbol
            ...DashboardTickerItemFragment_asset
          }
        }
      }
    `,
    {},
  );
  const assets = data.ticker?.nodes;

  return (
    <Ticker>
      {assets?.map((asset) => (
        <DashboardTickerItem key={asset.symbol} fragmentRef={asset} />
      ))}
    </Ticker>
  );
});
```

:::note

The example above is a teaser. Usually we want to have one or a few queries that accumulate all the data required to render the screen.

:::

Here how we can compose fragments at scale:

```jsx title="/scenes/dashboard/DashboardContainer.js"
import {Divider, Stack} from '@mui/material';
import ErrorPage from 'next/error';
import {memo} from 'react';
import {graphql, useLazyLoadQuery} from 'react-relay';

import DashboardFeatured from './DashboardFeatured';
import DashboardSpotlight from './DashboardSpotlight';
import DashboardTicker from './DashboardTicker';

export default memo(function DashboardContainer({cacheBuster}) {
  const data = useLazyLoadQuery(
    graphql`
      query DashboardContainerQuery {
        ...DashboardTickerFragment_query
        ...DashboardFeaturedFragment_query
        ...DashboardSpotlightFragment_query @defer(label: "spotlight")
      }
    `,
    {},
    {fetchKey: cacheBuster},
  );

  if (!data) {
    return <ErrorPage statusCode={404} title="Out of service" />;
  }

  return (
    <Stack gap={2}>
      <DashboardTicker fragmentRef={data} />
      <Divider />
      <DashboardFeatured fragmentRef={data} />
      <Divider />
      <DashboardSpotlight fragmentRef={data} />
    </Stack>
  );
});
```

```jsx title="/scenes/dashboard/DashboardTicker.js"
import {memo} from 'react';
import {graphql, useFragment} from 'react-relay';

import {Ticker} from '@/components';

import DashboardTickerItem from './DashboardTickerItem';

export default memo(function DashboardTicker({fragmentRef}) {
  const data = useFragment(
    graphql`
      fragment DashboardTickerFragment_query on Query {
        ticker: assets(
          first: 10
          order: {price: {tradableMarketCapRank: ASC}}
        ) {
          nodes {
            symbol
            ...DashboardTickerItemFragment_asset
          }
        }
      }
    `,
    fragmentRef,
  );
  const assets = data.ticker?.nodes;

  return (
    <Ticker>
      {assets?.map((asset) => (
        <DashboardTickerItem key={asset.symbol} fragmentRef={asset} />
      ))}
    </Ticker>
  );
});
```