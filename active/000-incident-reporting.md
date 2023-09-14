- Start Date: 2023-09-14
- RFC PR: https://github.com/datahub-project/rfcs/pull/5
- Discussion Issue: (GitHub issue this was discussed in before the RFC, if any)
- Implementation PR(s): (leave this empty)

# Dataset Incident Reporting

## Summary

> For DataHub user in our company, it has been a centralized place to discover and understand data. They also want 
> to use DataHub to report incidents related to Datasets. Especially it can potentially help to figure out all impacted 
> downstream Dataset owners. Thus, our proposal here is to add a new aspect of dataset: DatasetIncident. 

## Basic example

> The new aspect should be persisted in database for future auditing and UI display. The new aspect model might looks like:
> ```
> /**
> * Records dataset incident information
> */
> record DatasetIncident {
> /**
> * Query of the dataset incident
> */
> name: string
> /**
> * Slack link of the related incident
> */
> link: optional string
> /**
> * Description of the incident
> */
> description: string
> /**
> * Status of the incidents. e.g. Open, Closed, Deleted. Deleted mainly used for incident that is not valid, created by mistake and etc.
> */
> status: string
> /**
> * Timestamp of the incident
> */
> timestamp: long
> /**
> * Priority of this incident, p0 > p1 > p2.
> */
> priority: string
> /**
> * Reporter of the incident. Should be urn of users from DataHub.
> */
> reporter: string
> }
> ```

## Motivation

> Why are we doing this? What use cases does it support? What is the expected outcome?
>
> Major motivation is to provide a centralized place for users to figure out the incident impacted datasets. Ideally in 
> the lineage graph page. 

## Requirements

> What specific requirements does your design need to meet? This should ideally be a bulleted list of items you wish
> to achieve with your design. This can help everyone involved (including yourself!) make sure your design is robust
> enough to meet these requirements.
>
> 1. Allow authorized user to report/update/close incident of a dataset in Datahub UI/CLI/API. 
> 2. Incident Impacted datasets should display an icon around it's title as an indicator. 
> 3. Once a dataset is marked as incident, it's first level downstream dataset have warning indicator.
> 4. In lineage page, show above indicators as well. 
> 5. Send the first level downstream dataset core information(urn, owner) to reporter. 
> 6. Allow reporter to notify first level downstream dataset owners about the incident.

### Extensibility

> Please also call out extensibility requirements. Is this proposal meant to be extended in the future? Are you adding
> a new API or set of models that others can build on in later? Please list these concerns here as well.
> Yes, this requirement can be extended. For example the incident content can be rich to allow comments/announcement for 
> others to understand the progress of incident. Also potentially wire with assertions so that incident can be triggered 
> automatically.

## Non-Requirements

> Call out things you don't want to discuss in detail during this review here, to help focus the conversation. This can
> include things you may build in the future based off this design, but don't wish to discuss in detail, in which case
> it may also be wise to explicitly list that extensibility in your design is a requirement.
>
> This list can be high level and not detailed. It is to help focus the conversation on what you want to focus on.

## Detailed design

> This is the bulk of the RFC.

> Core of this design is to add DatasetIncident aspect to Dataset. 
> In UI side, we can follow similar fashion of tagging to allow user report incident. But we also want to have an incident tab in dataset page to show the incidents history of the dataset. 
> In BE, add incidentResolvers like taggingResolvers. Add new search logic to allow search historical incident and feed to UI for above purpose.
> For notification function, allow automatically send slack message/email based on the information from user profile. 

## How we teach this

> What names and terminology work best for these concepts and why? How is this idea best presented? As a continuation
> of existing DataHub patterns, or as a wholly new one?
> This is a continuation of existing DataHub patterns. 

> What audience or audiences would be impacted by this change? Just DataHub backend developers? Frontend developers?
> Users of the DataHub application itself?
> This will impact backend developers, frontend developers and users of the DataHub application itself.

> Would the acceptance of this proposal mean the DataHub guides must be re-organized or altered? Does it change how
> DataHub is taught to new users at any level?
> Yes, we need to add a new section in DataHub guides to teach users how to use this feature.

> How should this feature be introduced and taught to existing audiences?
> No clue yet. Need to revisit once the design is finalized.

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching DataHub, on the integration of this feature with
> other existing and planned features, on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Rollout / Adoption Strategy

> If we implemented this proposal, how will existing users / developers adopt it? Is it a breaking change? Can we write
> automatic refactoring / migration tools? Can we provide a runtime adapter library for the original API it replaces? 
> This is not a breaking change. 

## Future Work

> Describe any future projects, at a very high level, that will build off this proposal. This does not need to be
> exhaustive, nor does it need to be anything you work on. It just helps reviewers see how this can be used in the
> future, so they can help ensure your design is flexible enough.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still TBD?