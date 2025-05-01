# Adapting the DORA Score for Teams: A Guide to Measuring Software Delivery and Operational Performance Using Benchmarks
# Authored by Rob Beneson and Damon Gray

# Introduction

The composite DORA score of software delivery and operational
performance (SDO) is a metric that quantifies and allows for macro
comparison of benchmarked DevOps capabilities of any software
organization. It is based on the four key indicators of DORA: deployment
frequency, lead time for changes, change failure rate, and time to
restore service. These indicators reflect the speed, stability, and
reliability of the software delivery and operation processes.

*What are the problems we are trying to solve?*

1.  At the macro level, senior leaders have the proclivity to compare
    unlike services and try to drive improvement from that vantage
    point. Since many services are not apples to apples comparisons,
    this tactic bears little fruit and can cause directional and morale
    issues within and across teams.

2.  Not all teams can be compared against a single industry defined
    benchmark. Rather than compare against a single benchmark, teams
    should be compared against the benchmark that makes sense for the
    service they have created. As an example, a SOX service will have a
    very different profile than a backend API with non-confidential and
    unregulated data. Both can be highly effective but look very
    different for valid reasons.

# IDEA: Formula using benchmarks

The idea is that multiple benchmarks can be used to calculate a score.
There can be shared benchmarks and internal and/or historical service
benchmarks. A benchmark is described by a time component and a benchmark
value for each metric (DF, LT, CFR, TRS). Some of these benchmark values
are utilized as an average over the time or are used to compare each
data point that took place in the time and then take an average of the
deviation from the benchmark.

$$(\frac{DF}{BM_{DF}} - 1) + (1 - \ \frac{LT}{BM_{LT}}) + (\ 1 - \ \frac{TTRS}{BM_{TTRS}}) + (1 - \ \frac{CFR}{BM_{CFR}})\ 
$$

- See example data: [Download Example Scoring](https://github.com/robeneso_microsoft/AdaptedDORA/raw/main/DORA-example-scoring.xlsx)

- In the calculation if you hit the benchmark exactly then your score is
  0, if are better than the benchmark then your score is positive and if
  you are lagging the benchmark then it is negative.

- Below are how each numerator and denominator should be calculated. The
  denominator is basically the benchmark value. And each term is a
  percentage of the benchmark the service is hitting.

- $BM_{X}$ is a changeable benchmark for that metric, services can plug
  in their own benchmark (and can be based on their own data and goals)
  or based on industry norms like DORA.

## DF (Deployment Frequency)

DF seems like the easiest to calculate but is actually one of the more
nuanced as the bucketing of the values matters depending on where a team
is in their DevOps journey. We'll take a simple approach that will work
for any time frame and still have meaningful comparative usefulness and
easy scoring.

- Take the number of prod releases in time T and divide by number of
  days in time T

  - Recommendation to use T as a rolling 1 year or max time of service
    existence whichever is lowest.

  - Example: A team has completed 100 deployments in the last year, so
    their frequency is 100/365 = .274 deployments per day

  - Now there is an immediate issue with this as a team could do 100
    releases in a single month and none for the other 11 months. This
    can be solved by the more sophisticated calculation below. However,
    in aggregate these calculations are really to be able to see if a
    team is improving or against previous performance. Also, for a team
    that is delivering slowly in the past, can increase this score
    pretty quickly by deploying every day for 31 days, it would take it
    to 131/365 = .359 which is almost a 24% improvement. This simple
    approach would still have meaningful comparative benefits.

- Benchmarks would be for each of the 4 categories of DORA. For example:
  2 = Multiple times a day (basically on demand), .142 would be once a
  week, .033 would be once a month, etc.

  - Then any other benchmark could be used based on the team's previous
    frequency and compared year to year, month to month, etc.

  - It is also possible that 1 could be the highest benchmark as that
    would already assume more than 1 deployment a day because most teams
    do not deploy on Friday or the weekends.

## LT (Lead time to production)

Lead time could be from commit to production or PR to production,
however there are some complexities with this. Should it be when the
code was committed into the developers local repository or when it was
pushed to the server or when it was committed to the main branch or when
it was included in a PR. PR time to production also has some interesting
nuances as there are times when the PR to production could seem shorter
that reality if the PR was first from user branch -\> develop branch
took weeks, but then the develop branch -\> main branch took very little
time. There is also the difficulty of exactly what commits were going
into a release as squash commits become common for merging to main,
which effectively makes the time to production for that commit from the
instant of the completion of the PR. The goal is to get as close to the
true time for the velocity from code written to getting that value to
customers. However, unfortunately the industry has taken the "commit
time" as the time when it was committed to the main branch which could
be substantially later when the actual change was committed, and the
time taken to go through the stages of PR review (and possibly through
the stages of branch hierarchy if not working on trunk-based
development). To try to combat some of the downsides and inaccuracies
we'll take this novel approach.

- Each PR will take the time it was completed (usually these are only
  PRs to main) it will instead replace this value with the average age
  of all of its commits that made up that PR as it's time.

  - The easiest way to do this calculation is to take the average age of
    the commit's relative to the completed time of the PR and subtract
    that amount of time from the completed time of the PR. That is now
    considered the "time of commit" for the calculation of lead time.

  - Take note that a build/release could contain multiple PRs, so in
    those cases use a single time below for the deployment time, but you
    would do the calculation below for each PR in the time window
    between deployments. \[TODO: Add example\]

  - Example:

    - A PR is made up of 3 commits with the following times, 12/15/2023
      14:00, 12/18/2023 12:00, 11/25/2023 11:00

    - The PR was completed on 12/26/2023 11:00

    - The resultant changes from that PR were deployed to production on
      1/4/2024 16:00

    - Take all of the commit times and subtract them from the deployment
      time and take the average. For example, the 3 commits have a lead
      time of 20.08, 17.17, 40.21 respectively, which gives an average
      of 25.82 days. This is our new lead time.

- This new lead time calculation is then used to calculate an average
  for all of the PRs that have gone to production in the last T time
  (like in the last 30 or 60 days)

- The benchmarks would then be utilized to measure how the team is
  performing against indicators.

  - For example, the 4 benchmarks of DORA would be:

    - 1 hour for less than an hour

    - 24 hours for less than a day

    - 168 hours for less than a week

    - 720 hours for less than a month

## TTRS (Time to restore service)

Time to restore service is easy as it really only requires a single set
of "incidents" over a time period and if there is a good definition of
"impact start" and "mitigated time" is available. However, within
Microsoft there is a complexity of which incidents (usually IcM
incidents) should be accounted for. And to some extent whether a team or
service is utilizing IcM at all. One could take all incidents for a team
and calculate impact start -- time mitigated and be done with it.
However, many teams utilize things like Sev 4 incidents for
non-impacting incidents or even user requests. One might also take only
incidents marked with an Outage flag and utilize those. Also, TTRS is
often combined or at least has a corollary to change failure rate
(below) and there could be an argument that they utilize the same set of
incidents and with this it is often the best source of date for both
would be any incident with a post mortem (aka retrospective), as these
have the best reviewed data as far as impact start and remediation time
as well as whether the outage was related to change. Given this the more
accurate list of "incidents" should be preferred and used for both TTRS
and CFR.

- Take all incidents that have a retrospective.

- Take the elapsed time from "Impact start" to "Mitigation" as the time
  to restore.

- To remove huge outliers, it is recommended to take the trimmed or
  truncated mean of this time in hours.

  - Suggestion would be to do a trimmed mean of 5% or even the
    interquartile mean (remove lowest and highest 25%) and then take the
    average in hours.

- Benchmarks would be 1, 24, 168, 720 (just like DORA, although these
  should be adjusted as necessary)

## CFR (Change Failure Rate)

Change failure rate is somewhat easy to calculate if the "outage"
information is easy to retrieve and tie back to a specific release. This
is obviously easier said than done. As mentioned above for TTRS, it
should be considered to use the same data source as TTRS for CFR,
meaning take only incidents that have a retrospective.

- Take all incidents that have a retrospective and that are marked with
  "Caused by Change" = Yes over time T (In Kusto IsCausedByChange ==
  true)

- Take the number of deployments to production (preferably the same set
  used in the calculation of lead time above) in the same time T

- $\frac{(Total\ Incidents\ Caused\ by\ Change)}{Total\ Changes\ To\ Production}*100.0$

- This will give a percentage of changes that caused an outage.

- This of course has some inaccuracies as it doesn't take into account a
  single change causing more than one outage or a change that was made
  manually outside of the dev pipeline.

- Benchmarks would be 15% & 30%.

# Terms and Definitions

- DF = Deployment Frequency: The number of times per unit of time (e.g.,
  day, week, month) that an organization successfully releases software
  to production. A higher deployment frequency indicates a faster and
  more frequent delivery of value to customers. (*Logarithmic scale for
  Deployment Frequency*)

- LT = Lead Time for Changes: The amount of time elapsed from the moment
  a code change is committed to the version control system until it is
  deployed to production. A lower lead time for changes indicates a
  shorter and more efficient delivery cycle. *(Inverse scale for Lead
  Time)*

- CFR = Change Failure Rate: The percentage of deployments that result
  in a failure in production, such as service impairment, outage,
  rollback, or hotfix. A lower change failure rate indicates higher
  quality and reliability of the software and the delivery process.
  *(Exponential decay for Change Failure Rate)*

- TRS = Time to Restore Service: The amount of time elapsed from the
  moment a failure in production is detected until it is resolved, and
  the service is restored to normal operation. A lower time to restore
  service indicates a faster and more effective recovery from incidents.
  *(Exponential decay for Time to Restore Service)*
