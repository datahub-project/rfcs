- Start Date: (fill me in with today's date, 2023-06-22)
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Discussion Issue: (GitHub issue this was discussed in before the RFC, if any)
- Implementation PR(s): (leave this empty)

# Allow OpenAPI Ingest To Generate Any Method Type With Example

## Summary

> Currently, the OpenAPI ingest only acts on endpoint that utilize GETs. If the OAS adequately defines the example data from the endpoint, Datahub should be able to ingest any method type as it does not need to interact with the endpoint.

## Basic example

> The example needed in the OpenAPI spec is on the response data, as in https://swagger.io/docs/specification/adding-examples/
>
> This can look like
> ```
> "responses": {
>   "200": {
>     "content": {
>       "application/json": {
>         "example": {
>           "foo": "a string",
>           "bar": 12
>         }
>       }
>     }
>   }
> }
> ```
> With this provided, Datahub will process the field name and datatype.

## Motivation

> This change is to allow the OAS ingestion to cover more endpoints and provide more metadata for APIs.  This supports cases where the API definition contains multiple supported endpoint methods, so long as they meet the minimum criteria of providing a data example.
>
> The expected outcome is that all adequately specified APIs are represented in Datahub.

## Requirements

> * Do not filter on GET-only in `openapi_parser.py/get_endpoints`
> * Store the method for each endpoint so that it can be evaluated when getting workunits
> * When getting workunits, read the example data if provided for any endpoint.  Skip non-GETs if there is no example

### Extensibility

> N/A

## Non-Requirements

> It is possible there would be future ability via some other flags or data that would allow searching other methods for data (making sample calls as it presently does for GET), so it should be made extensible enough to support that.

## Detailed design

> In `openapi_parser.py/get_endpoints`, the method type can be obtained from the first key in the dict, ie `method = list(p_o)[0]` [line 124](https://github.com/datahub-project/datahub/blob/620d245d570dd251713a89c33adea106560267b1/metadata-ingestion/src/datahub/ingestion/source/openapi_parser.py#L122).  Remove the `if "get"` line, and store the method in the dict `url_details[p_k] = {"description": desc, "tags": tags, "method": method}`.
>
> Then in `openapi.py/get_workunits_internal` where the above dict gets used, add an
> 
> ```
> elif endpoint_dets["method"] != "get":
>     continue
> ```
> after line [252](https://github.com/datahub-project/datahub/blob/620d245d570dd251713a89c33adea106560267b1/metadata-ingestion/src/datahub/ingestion/source/openapi.py#L252)
>
> Additional error catching should be added so that the user is informed when an endpoint is not included because it is missing example data.

## How we teach this

> This doesn't have widespread impact outside of OpenAPI ingest recipes.  For these, it allows for ingestion of all method types. It needs to noted that this will _only_ work if example is provided

## Drawbacks

> This requires a little more education, and may lead to confusion on why some POST methods are included and others aren't, for example.

## Alternatives

> No other alternative was considered, this was a quick fix I made locally to test getting my other endpoint ingested by Datahub

## Rollout / Adoption Strategy

> This is not a breaking change, though users will potentially want to re-run and OpenAPI recipes they have, or update their existing specs to include more example data.

## Future Work

> Unknown

## Unresolved questions

> The small changes above worked in my test environment.  Existing questions remain about if there were additional reasons that I'm unaware of, besides the danger and complexity of interacting with other methods, that this was explicitly left out previously