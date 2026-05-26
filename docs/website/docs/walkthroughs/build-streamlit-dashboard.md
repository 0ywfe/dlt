---
title: Build a Streamlit dashboard
description: Build, serve, and deploy a Streamlit app on dltHub.
keywords: [streamlit, dashboard, hub, app, deploy, dltHub]
---

# Build a Streamlit dashboard

[Streamlit](https://docs.streamlit.io/) is a Python framework for turning a script into an interactive web app. On dltHub, a Streamlit app is a plain `.py` file that imports `streamlit`, and the runtime serves it as an interactive dashboard.

This page walks through building a small dashboard against a loaded dlt dataset and deploying it to [app.dlthub.com](https://app.dlthub.com).

## Prerequisites

Add Streamlit to your workspace dependencies:

```sh
uv add streamlit
```

The example below reads from the `starter_pipeline` that the `dlthub-start` scaffold ships (Open Brewery DB → `warehouse` destination, `brewery_data` dataset). The dashboard needs that data already loaded against the same destination it'll read from:

```sh
# Load locally (dev profile, DuckDB) so the dashboard works in `streamlit run`
uv run dlthub local run load_breweries

# OR, before deploying remotely, load against the prod destination on dltHub
uv run dlthub run load_breweries
```

A dashboard can only display data that's already been loaded. If you skip this step the deployed app boots but every read returns "table not found".

## Write the dashboard

Create `breweries_dashboard.py` in the workspace root:

```python
"""Breweries dashboard."""

import dlt
import streamlit as st


st.set_page_config(page_title="Breweries", layout="wide")
st.title("US breweries")


@st.cache_data
def load_breweries():
    dataset = dlt.dataset(destination="warehouse", dataset_name="brewery_data")
    return dataset["breweries"].df()


breweries = load_breweries()

col1, col2, col3 = st.columns(3)
col1.metric("Breweries", f"{len(breweries):,}")
col2.metric("States", breweries["state"].nunique())
col3.metric("Distinct types", breweries["brewery_type"].nunique())

states = sorted(breweries["state"].dropna().unique().tolist())
picked = st.multiselect("Filter by state", states, default=[])
filtered = breweries[breweries["state"].isin(picked)] if picked else breweries

st.dataframe(
    filtered[["name", "brewery_type", "city", "state", "website_url"]],
    width="stretch", hide_index=True,
)

by_type = filtered.groupby("brewery_type", dropna=False).size().sort_values(ascending=False)
st.bar_chart(by_type.rename("count"))
```

[`dlt.dataset(destination, dataset_name)`](../general-usage/dataset-access/dataset.md) connects directly to a loaded dataset. Wrap the call in `@st.cache_data` so each widget interaction doesn't requery the destination.

## Run it locally

```sh
uv run dlthub local serve breweries_dashboard.py
```

This boots the dashboard under the workspace's active local profile (default `dev`, which reads from `.dlt/dev.config.toml`) and opens it in your browser.

## Configure the `access` profile

`dlthub serve` runs interactive jobs under the `access` profile by default. Configure the destination type in `.dlt/access.config.toml`:

```toml
[destination.warehouse]
destination_type = "motherduck"
```

And the credentials in `.dlt/access.secrets.toml`. **Both `database` and `password` need to be in the secrets file** for MotherDuck:

```toml
[destination.warehouse.credentials]
database = "dlt_test"
password = "<read-only motherduck JWT>"
```

See the [Profiles in dltHub](../hub/core-concepts/profiles-dlthub.md) page for the full profile model.

## Deploy to dltHub

Add the dashboard to your `__deployment__.py` manifest so the workspace knows about it:

```python
"""Walkthrough deployment manifest."""

from starter_pipeline import load_breweries
import breweries_dashboard            # module-import → one job

__all__ = ["load_breweries", "breweries_dashboard"]
```

Then deploy and serve:

```sh
uv run dlthub deploy                                   # publishes manifest + uploads code
uv run dlthub serve breweries_dashboard.py             # boots the app remotely, opens URL
```

`dlthub serve` runs the app behind the workspace's auth — only your account can open the link. To create a publicly shareable URL:

```sh
uv run dlthub job publish breweries_dashboard.py       # public URL
uv run dlthub job unpublish breweries_dashboard.py     # revoke
```
