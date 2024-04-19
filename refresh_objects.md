# Refreshing Workbooks, Datasources, and Flows

<!--toc:start-->
- [Refreshing Workbooks, Datasources, and Flows](#refreshing-workbooks-datasources-and-flows)
  - [Finding the ID of the Item to Refresh](#finding-the-id-of-the-item-to-refresh)
  - [Starting the Refresh](#starting-the-refresh)
  - [Monitoring the Refresh](#monitoring-the-refresh)
  - [Conclusion](#conclusion)
<!--toc:end-->

Scheduling refreshes of items on Tableau Server is critical to make sure that
the data you see is up to date. This can become a problem when you need to
time that refresh to occur after another process has completed, like the ETL
jobs that populate the base tables. Fortunately, the Tableau REST API makes it
simple to trigger a refresh of a workbook, datasource, or flow.



## Finding the ID of the Item to Refresh

In this example, I am going to sign in using a personal access token. If you
don't know how to set that up, check out the Authentication section of the
[Download From View Name](download_from_view_name.md) page.

```python
import logging
import os

from dotenv import load_dotenv
import tableauserverclient as TSC

load_dotenv()
logger = logging.getLogger(__name__)

server = TSC.Server(os.getenv("TABLEAU_SERVER"), True)
auth = TSC.PersonalAccessTokenAuth(
    token_name=os.getenv("TABLEAU_PAT_NAME"),
    personal_access_token=os.getenv("TABLEAU_PAT_SECRET"),
    # For site_id, use the site content URL, not the site's LUID. See
    # the example screenshot below.
    site_id="example-site-id"
)

with server.auth.sign_in(auth):
    ...
```

To refresh an item, you need to know the ID of the item. All of the refresh
methods will either accept the id directly, or the TSC Item object. Let's say
I have a variety of datasources, flows, and workbooks that I need to refresh
in this same script. Thankfully, all of these items can utilize the
[Django style filters](https://tableau.github.io/server-client-python/docs/filter-sort#django-style-filters-and-sorts)
to find the item I need.

First, let's find the ID of the items that need to be refreshed. I don't know
these off hand, but I do know some attributes of the items that I can use to
filter down to the item I need. First I'll build a collection of the item types
and attributes I need to filter on. I will create a dictionary of the
refreshable items, with the filters I need to find the item.

```python

# The collection of items I need to refresh
refresh_items = {
    "datasources": [
        {"name": "Sales Data", "owner_email": "John.Doe@example.com"},
        {"name": "Product Data", "tags": "special"},
    ],
    "workbooks": [
        {"name": "Sales Dashboard", "project_name": "Marketing"},
        {"name": "Inventory Dashboard", "owner_name": "Jane Doe", "project_name": "Supply Chain"},
    ],
    "flows": [
        {"name": "Marketing funnel", "owner_name": "John Doe"},
        {"name": "Month End", "project_name": "Finance"},
    ],
}
```

Then I will create a helper function to find the item based on the filters
above. TSC conveniently follows the same pattern for each endpoint, so we can
leverage the dynamic nature of python to grab the endpoint we need on demand.
Since TSC doesn't have a top level type all items inherit from, I will make a
protocol to make the type hinting easier.

```python
from typing import Literal, Protocol
refreshable = Literal["datasources", "workbooks", "flows"]

class HasID(Protocol):
    id: str


def get_item(
    server: TSC.Server,
    item_type: refreshable,
    filters: dict[str, str],
) -> HasID:
    endpoint = getattr(server, item_type)
    query_result = endpoint.filter(**filters)
    if (count := len(query_result)) != 1:
        raise ValueError(f"Expected 1 {item_type} to match filters, found {count}")

    return query_result[0]

```

Or perhaps you want to refresh all items that match the filters. A small tweak
makes that simple.

```python
from typing import Iterator

def get_items(server: TSC.Server, item_type: refreshable, filters: dict) -> Iterator[HasID]:
    endpoint = getattr(server, item_type)
    query_result = endpoint.filter(**filters)
    return query_result
```

## Starting the Refresh

Now I craft another helper function to refresh the item, again leveraging the
similar pattern of the endpoints and the dynamic nature of python.

```python

def refresh_item(
    server: TSC.Server,
    item_type: refreshable,
    item: HasID
) -> TSC.JobItem:
    endpoint = getattr(server, item_type)
    return endpoint.refresh(item)
```

Now I can loop through the items, trigger the refreshes, and collect the job
items.

```python

jobs = []

with server.auth.sign_in(auth):
    for item_type, items in refresh_items.items():
        for filters in items:
            item = get_item(server, item_type, filters)
            job = refresh_item(server, item_type, item)
            logger.info(f"Started refresh for {item_type} {item.name} with job id {job.id}")
            jobs.append(job)
```

## Monitoring the Refresh

Now optionally, I may want to wait for the jobs to complete. Since I need to
wait for all of the jobs to complete, there is no need to get clever with
checking how many jobs are still running, so we can just wait for them naively.
However, we will need to handle cases where the refresh has either failed or
has been canceled.

```python
from tableauserverclient.server.exceptions import JobCancelledException, JobFailedException 

def wait_for_job(server: TSC.Server, job: TSC.JobItem) -> TSC.JobItem:
    try:
        return server.jobs.wait_for_job(job)
    except (JobCancelledException, JobFailedException) as e:
        logger.exception(f"Job {job} failed or was canceled")
    return server.jobs.get_by_id(job.id)

jobs = [wait_for_job(server, job) for job in jobs]

```

## Conclusion

And that's it! You've now refreshed all of the items you needed to. If you need
to do any additional handling of job state, you can add it to the `wait_for_job`
function, or after the loop of waiting for jobs.

Putting it all together, we have the following script:

```python
# /// script
# requires-python = ">=3.8"
# dependencies = [
#    "tableauserverclient",
#    "python-dotenv"
# ]
# ///

import logging
import os
from typing import Literal, Protocol

from dotenv import load_dotenv
import tableauserverclient as TSC
from tableauserverclient.server.exceptions import JobCancelledException, JobFailedException 

load_dotenv()
logger = logging.getLogger(__name__)

refreshable = Literal["datasources", "workbooks", "flows"]

class HasID(Protocol):
    id: str


server = TSC.Server(os.getenv("TABLEAU_SERVER"), True)
auth = TSC.PersonalAccessTokenAuth(
    token_name=os.getenv("TABLEAU_PAT_NAME"),
    personal_access_token=os.getenv("TABLEAU_PAT_SECRET"),
    # For site_id, use the site content URL, not the site's LUID. See
    # the example screenshot below.
    site_id="example-site-id"
)
# The collection of items I need to refresh
refresh_items = {
    "datasources": [
        {"name": "Sales Data", "owner_email": "John.Doe@example.com"},
        {"name": "Product Data", "tags": "special"},
    ],
    "workbooks": [
        {"name": "Sales Dashboard", "project_name": "Marketing"},
        {"name": "Inventory Dashboard", "owner_name": "Jane Doe", "project_name": "Supply Chain"},
    ],
    "flows": [
        {"name": "Marketing funnel", "owner_name": "John Doe"},
        {"name": "Month End", "project_name": "Finance"},
    ],
}

def get_item(
    server: TSC.Server,
    item_type: refreshable,
    filters: dict[str, str],
) -> HasID:
    endpoint = getattr(server, item_type)
    query_result = endpoint.filter(**filters)
    if (count := len(query_result)) != 1:
        raise ValueError(f"Expected 1 {item_type} to match filters, found {count}")

    return query_result[0]


def refresh_item(
    server: TSC.Server,
    item_type: refreshable,
    item: HasID
) -> TSC.JobItem:
    endpoint = getattr(server, item_type)
    return endpoint.refresh(item)


def wait_for_job(server: TSC.Server, job: TSC.JobItem) -> TSC.JobItem:
    try:
        return server.jobs.wait_for_job(job)
    except (JobCancelledException, JobFailedException) as e:
        logger.exception(f"Job {job} failed or was canceled")
    return server.jobs.get_by_id(job.id)


jobs = []
def main() -> None:
    with server.auth.sign_in(auth):
        for item_type, items in refresh_items.items():
            for filters in items:
                item = get_item(server, item_type, filters)
                job = refresh_item(server, item_type, item)
                logger.info(f"Started refresh for {item_type} {item.name} with job id {job.id}")
                jobs.append(job)

        jobs = [wait_for_job(server, job) for job in jobs]

if __name__ == "__main__":
    main()
```

