# Aggregate Sampling
Make sampling decisions by applying a logical operator (AND/OR) to the outcome of multiple other samplers.

## Motivation
This change will allow Open Telemetry to address more complex sampling scenarios by:
* Allowing the implementation of smaller, more discrete, samplers that are easier to maintain and to understand.
* Allowing implementations to blend multiple sampling approaches.
* Allowing for the componentization of sampling logic into smaller blocks so that a customer may choose which samplers to include in their configuration.
* Allowing the customer to determine the order in which the samplers are evaluated.
* To better support complex scenarios.

## Explanation
Aggregate samplers evaluate the results of other samplers, known as inner samplers and applying a logical operator to obtain a result.  Inner samplers are instantiated during configuration.  There are two types of Aggregate Samplers:

##### AndSampler
The `AndSampler` executes all of the inner samplers and provides a `ShouldSample` result of `true` when *all* of the inner samplers have returned a result of `true`.

```C#
    var myFilteringOutNoiseSampler = new AndSampler
    (
        new IgnoredTraceCustomSampler(),
        new PrioritySampler()
    );
```

In the example above, both the `IgnoredTraceCustomSampler` and the `PrioritySampler` must return `true` for this activity to be sampled. 

##### OrSampler
The `OrSampler` executes all of the inner samplers and provides a `ShouldSample` result of `true` when *any* of the inner samplers have returned a result of `true`.

```C#
    var myReportingImportantThingsSampler = new OrSampler
    (
        new KeyTraceCustomSampler(),
        new PrioritySampler()
    );
```

In the example above, either the `KeyTraceCustomSampler` or the `PrioritySampler` must return `true` for this activity to be sampled. 


##### Nesting Samplers for Complex Scenarios
Aggregate Samplers are samplers.  As such, they may be nested to implement complex scenarios.

```
    var myImportantThingsButNotTooNoisySampler = new OrSampler
    (
       new KeyTraceCustomSampler(),
       new AndSampler
       (
            new IgnoreTraceCustomSampler(),
            new PrioritySampler()
       )
    );
```

## Internal details

This would be implemented by building two new samplers, `AndSampler` and `OrSampler`, that are implementations of the `Sampler` class.  
* Constructors would accept 2..n samplers.
* The `ShouldSample` method would apply the logical operator to the results to determine the overall `ShouldSample` Result.
* The evaluation of the sampers should occur in the order in which they were specified in the constructor.


##### Integrating with existing functionality
This change would co-exist with existing functionality because the `AndSampler` and the `OrSampler` are samplers.

This change could simplify the `ParentOrElseSampler` to just be a sampler that evaluates if the Parent Trace has been sampled.

##### Errors
Any exceptions that occur during the execution of an inner sampler should be collected and bubbled up as an `AggregateException`.

## Trade-offs and mitigations
  
##### Determining why an activity was included/excluded  
With multiple samplers executing and their results being aggregated, it may be difficult to understand why something was sampled or not sampled.  The sampler that caused an activity to be sampled may be important to an observability platform.

For example, An a `KeyTraceSampler` may be developed which forces the sampling of "important" traces.  When the observability platform receives an activity that is "important", it may need to do something specific, such as perform a notification/alert.  

One way that this could be mitigated may be to allow Samplers to update the activity (or append to it) with the outcomes of their inner samplers. 

## Prior art and alternatives

* Creating `AndSampler` and `OrSampler` as custom samplers.
* Relying on the creation of custom samplers to accomplish the specific task.

## Open questions

* Should the sampler be able to update the Activity to add more detail about how the sampling decision was arrived at?

## Future possibilities

What are some future changes that this proposal would enable?
