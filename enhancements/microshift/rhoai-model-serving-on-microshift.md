---
title: rhoai-model-serving-on-microshift
authors:
  - pmtk
reviewers: # Include a comment about what domain expertise a reviewer is expected to bring and what area of the enhancement you expect them to focus on. For example: - "@networkguru, for networking aspects, please look at IP bootstrapping aspect"
  - Danielf - MicroShift PM
  - Geri - MicroShift Staff Eng, Architect
  - Someone from RHOAI
approvers: # A single approver is preferred, the role of the approver is to raise important questions, help ensure the enhancement receives reviews from all applicable areas/SMEs, and determine when consensus is achieved such that the EP can move forward to implementation.  Having multiple approvers makes it difficult to determine who is responsible for the actual approval.
  - Geri
api-approvers:
  - None
creation-date: 2025-01-10
last-updated: 2025-01-10
tracking-link:
  - https://issues.redhat.com/browse/OCPSTRAT-1721
# see-also:
#   - "/enhancements/this-other-neat-thing.md"
# replaces:
#   - "/enhancements/that-less-than-great-idea.md"
# superseded-by:
#   - "/enhancements/our-past-effort.md"
---

# RHOAI Model Serving on MicroShift

## Summary

Following enhancement describes process of enabling AI models
serving on MicroShift that is based on Red Hat OpenShift AI (RHOAI).

## Motivation

Enabling users to use MicroShift for AI model serving means they will be able to
train model in the cloud or datacenter on OpenShift, and serve models at the
edge using MicroShift.

### User Stories

* As a MicroShift user, I want to serve AI models on the edge in a lightweight manner.

### Goals

- Prepare RHOAI based kserve manifests that fit MicroShift's use cases and environments.
- Provide RHOAI supported ServingRuntimes CRs so that users can use them.
  - [List of supported model-serving runtimes](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/2.16/html/serving_models/serving-large-models_serving-large-models#supported-model-serving-runtimes_serving-large-models) (not all might be suitable for MicroShift)
- Document how to use kserve on MicroShift.

### Non-Goals

- Deploying full RHOAI on MicroShift
- Providing goodies such as RHOAI Dashboard. It will be more similar to upstream kserve usage.

## Proposal

Extract kserve manifests from RHOAI Operator image and adjust them for MicroShift:
- Make sure that cert-manager is not required, instead leverage OpenShift's service-ca.
  - This might require adding some extra annotations to resources so the service-ca injects the certs.
- Drop requirement for Istio as an Ingress Controller and use OpenShift Router instead.
  - Done by changing kserve's setting configmap to use another ingress controller.
- Use 'RawDeployment' mode, so that neither Service Mesh nor Serverless are required,
  to minimize to make the solution suitable for edge devices.
  - Also done in the configmap.
- Package the manifests as an RPM `microshift-kserve` for usage.

Provide users with ServingRuntime definitions derived from RHOAI, so they
are not forced to use upstream manifests.
Decision on how to do this is pending. See open questions.

### Workflow Description

**User** is a human administrating and using MicroShift cluster/device.

(RPM vs ostree vs bootc is skipped because it doesn't differ from any other MicroShift's RPM).

1. User installs `microshift-kserve` RPM and restarts MicroShift service.
1. Kserve manifest are deployed.
1. ServingRuntimes are delivered with the kserve RPM or deployed by the user.
1. User creates InferenceService CR which references ServingRuntime of their choice
   and reference to the model.
1. Kserve creates Deployment, Ingress, and other.
1. Resources from previous step become ready and user can make HTTP/GRPC calls
   to the model server.

### API Extensions

`microshift-kserve` RPM will bring following CRDs, however they're not becoming
part of the core MicroShift deployment:
- InferenceServices
- TrainedModels
- ServingRuntimes
- InferenceGraphs
- ClusterStorageContainers
- ClusterLocalModels
- LocalModelNodeGroups

Contents of these CRDs can be viewed at https://github.com/red-hat-data-services/kserve/tree/master/config/crd/full.

### Topology Considerations

#### Hypershift / Hosted Control Planes

Enhancement is MicroShift specific.

#### Standalone Clusters

Enhancement is MicroShift specific.

#### Single-node Deployments or MicroShift

Enhancement is MicroShift specific.

### Implementation Details/Notes/Constraints

-

### Risks and Mitigations

RHOAI documentation states:

> Deploying a machine learning model using KServe raw deployment mode on single
> node OpenShift is a Limited Availability feature. Limited Availability means
> that you can install and receive support for the feature only with specific
> approval from the Red Hat AI Business Unit. Without such approval, the feature
> is unsupported

This one needs to be solved at Product Management level: whether all MicroShift
customers get support exception or we indefinally make this "developer previous"
with whatever little support MicroShift team can provide.

### Drawbacks

- Only x86_64 architecture is supported.

## Open Questions [optional]

### Tweaking Kserve settings

Kserve settings are delivered in form of a ConfigMap:
- [Upstream example](https://github.com/red-hat-data-services/kserve/blob/master/config/configmap/inferenceservice.yaml)
- [RHOAI's overrides](https://github.com/red-hat-data-services/kserve/blob/master/config/overlays/odh/inferenceservice-config-patch.yaml)

While we can recommend users to create a manifest that will override the
ConfigMap to their liking, there's one setting that we could handle better:
`ingress.ingressDomain`.  In the example it has value of `example.com` and it
might be poor UX to require every customer to create a new ConfigMap just to change this.

Possible solution is to change our manifest handling logic, so that MicroShift 
uses kustomize's Go package to first render the manifest, then template it,
and finally apply. For this particular `ingress.ingressDomain` we could reuse value of
`dns.baseDomain` from MicroShift's config.yaml.


### How to deliver ServingRuntime CRs

From [kserve documentation](https://kserve.github.io/website/master/modelserving/servingruntimes/):

> KServe makes use of two CRDs for defining model serving environments:
> 
> ServingRuntimes and ClusterServingRuntimes
> 
> The only difference between the two is that one is namespace-scoped and the other is cluster-scoped.
> 
> A ServingRuntime defines the templates for Pods that can serve one or more 
> particular model formats. Each ServingRuntime defines key information such as
> the container image of the runtime and a list of the model formats that the
> runtime supports. Other configuration settings for the runtime can be conveyed
> through environment variables in the container specification.

RHOAI approach to ServingRuntimes:
- ClusterServingRuntimes are not supported (the CRD is not created).
- Each usable ServingRuntime is wrapped in Template and resides in RHOAI's namespace.
- When user uses RHOAI Dashboard to serve a model, they must select a runtime
  from a list (which is constructed from Templates holding ServingRuntime)
  and provide information about the model.
- When the user sends the form, Dashboard creates a ServingRuntime from the
  Template in the user's Data Science Project (effectively Kubernetes namespace),
  and assembles the InferenceService CR.

Problem: how to provide user with ServingRuntimes without creating unnecessary obstacles.

In any of the following solutions, we need to drop the `Template` container to
get only the `ServingRuntime` CR part.

Potential solutions so far:
- Include ServingRuntimes in a RPM as a kustomization manifest (might be 
  `microshift-kserve`, `microshift-rhoai-runtimes`, or something else) using a
  specific predefined namespace.
  - This will force users to either use that namespace for serving, or they can
    copy the SRs to their namespace (either at runtime, or by including it in
    their manifests).
- Change ServingRuntimes to ClusterServingRuntimes and include them in an RPM,
  so they're accessible from any namespace (MicroShift is intended for
  single-user operation anyway).
- Don't include SRs in any of the RPMs. Instead include them in the documentation
  for users to copy and include in their manifests.
  - This might be prone to getting outdated very easily as documentation is not
    part of the MicroShift rebase procedure.


### Versioning

TODO: RHOAI does not follow OpenShift's release schedule.


## Test Plan

<!-- **Note:** *Section not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?
- What additional testing is necessary to support managed OpenShift service-based offerings?

No need to outline all of the test cases, just the general strategy. Anything
that would count as tricky in the implementation and anything particularly
challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations). -->

## Graduation Criteria

<!-- **Note:** *Section not required until targeted at a release.*

Define graduation milestones.

These may be defined in terms of API maturity, or as something else. Initial proposal
should keep this high-level with a focus on what signals will be looked at to
determine graduation.

Consider the following in developing the graduation criteria for this
enhancement:

- Maturity levels
  - [`alpha`, `beta`, `stable` in upstream Kubernetes][maturity-levels]
  - `Dev Preview`, `Tech Preview`, `GA` in OpenShift
- [Deprecation policy][deprecation-policy]

Clearly define what graduation means by either linking to the [API doc definition](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-versioning),
or by redefining what graduation means.

In general, we try to use the same stages (alpha, beta, GA), regardless how the functionality is accessed.

[maturity-levels]: https://git.k8s.io/community/contributors/devel/sig-architecture/api_changes.md#alpha-beta-and-stable-versions
[deprecation-policy]: https://kubernetes.io/docs/reference/using-api/deprecation-policy/

**If this is a user facing change requiring new or updated documentation in [openshift-docs](https://github.com/openshift/openshift-docs/),
please be sure to include in the graduation criteria.**

**Examples**: These are generalized examples to consider, in addition
to the aforementioned [maturity levels][maturity-levels]. -->

### Dev Preview -> Tech Preview

<!-- - Ability to utilize the enhancement end to end
- End user documentation, relative API stability
- Sufficient test coverage
- Gather feedback from users rather than just developers
- Enumerate service level indicators (SLIs), expose SLIs as metrics
- Write symptoms-based alerts for the component(s) -->

### Tech Preview -> GA

<!-- - More testing (upgrade, downgrade, scale)
- Sufficient time for feedback
- Available by default
- Backhaul SLI telemetry
- Document SLOs for the component
- Conduct load testing
- User facing documentation created in [openshift-docs](https://github.com/openshift/openshift-docs/)

**For non-optional features moving to GA, the graduation criteria must include
end to end tests.** -->

### Removing a deprecated feature

<!-- - Announce deprecation and support policy of the existing feature
- Deprecate the feature -->

## Upgrade / Downgrade Strategy

<!-- If applicable, how will the component be upgraded and downgraded? Make sure this
is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this
enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to keep previous behavior?
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade in order to make use of the enhancement?

Upgrade expectations:
- Each component should remain available for user requests and
  workloads during upgrades. Ensure the components leverage best practices in handling [voluntary
  disruption](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/). Any exception to
  this should be identified and discussed here.
- Micro version upgrades - users should be able to skip forward versions within a
  minor release stream without being required to pass through intermediate
  versions - i.e. `x.y.N->x.y.N+2` should work without requiring `x.y.N->x.y.N+1`
  as an intermediate step.
- Minor version upgrades - you only need to support `x.N->x.N+1` upgrade
  steps. So, for example, it is acceptable to require a user running 4.3 to
  upgrade to 4.5 with a `4.3->4.4` step followed by a `4.4->4.5` step.
- While an upgrade is in progress, new component versions should
  continue to operate correctly in concert with older component
  versions (aka "version skew"). For example, if a node is down, and
  an operator is rolling out a daemonset, the old and new daemonset
  pods must continue to work correctly even while the cluster remains
  in this partially upgraded state for some time.

Downgrade expectations:
- If an `N->N+1` upgrade fails mid-way through, or if the `N+1` cluster is
  misbehaving, it should be possible for the user to rollback to `N`. It is
  acceptable to require some documented manual steps in order to fully restore
  the downgraded cluster to its previous state. Examples of acceptable steps
  include:
  - Deleting any CVO-managed resources added by the new version. The
    CVO does not currently delete resources that no longer exist in
    the target version. -->

## Version Skew Strategy

<!-- How will the component handle version skew with other components?
What are the guarantees? Make sure this is in the test plan.

Consider the following in developing a version skew strategy for this
enhancement:
- During an upgrade, we will always have skew among components, how will this impact your work?
- Does this enhancement involve coordinating behavior in the control plane and
  in the kubelet? How does an n-2 kubelet without this feature available behave
  when this feature is used?
- Will any other components on the node change? For example, changes to CSI, CRI
  or CNI may require updating that component before the kubelet. -->

## Operational Aspects of API Extensions

<!-- Describe the impact of API extensions (mentioned in the proposal section, i.e. CRDs,
admission and conversion webhooks, aggregated API servers, finalizers) here in detail,
especially how they impact the OCP system architecture and operational aspects.

- For conversion/admission webhooks and aggregated apiservers: what are the SLIs (Service Level
  Indicators) an administrator or support can use to determine the health of the API extensions

  Examples (metrics, alerts, operator conditions)
  - authentication-operator condition `APIServerDegraded=False`
  - authentication-operator condition `APIServerAvailable=True`
  - openshift-authentication/oauth-apiserver deployment and pods health

- What impact do these API extensions have on existing SLIs (e.g. scalability, API throughput,
  API availability)

  Examples:
  - Adds 1s to every pod update in the system, slowing down pod scheduling by 5s on average.
  - Fails creation of ConfigMap in the system when the webhook is not available.
  - Adds a dependency on the SDN service network for all resources, risking API availability in case
    of SDN issues.
  - Expected use-cases require less than 1000 instances of the CRD, not impacting
    general API throughput.

- How is the impact on existing SLIs to be measured and when (e.g. every release by QE, or
  automatically in CI) and by whom (e.g. perf team; name the responsible person and let them review
  this enhancement)

- Describe the possible failure modes of the API extensions.
- Describe how a failure or behaviour of the extension will impact the overall cluster health
  (e.g. which kube-controller-manager functionality will stop working), especially regarding
  stability, availability, performance and security.
- Describe which OCP teams are likely to be called upon in case of escalation with one of the failure modes
  and add them as reviewers to this enhancement. -->

## Support Procedures

<!-- Describe how to
- detect the failure modes in a support situation, describe possible symptoms (events, metrics,
  alerts, which log output in which component)

  Examples:
  - If the webhook is not running, kube-apiserver logs will show errors like "failed to call admission webhook xyz".
  - Operator X will degrade with message "Failed to launch webhook server" and reason "WehhookServerFailed".
  - The metric `webhook_admission_duration_seconds("openpolicyagent-admission", "mutating", "put", "false")`
    will show >1s latency and alert `WebhookAdmissionLatencyHigh` will fire.

- disable the API extension (e.g. remove MutatingWebhookConfiguration `xyz`, remove APIService `foo`)

  - What consequences does it have on the cluster health?

    Examples:
    - Garbage collection in kube-controller-manager will stop working.
    - Quota will be wrongly computed.
    - Disabling/removing the CRD is not possible without removing the CR instances. Customer will lose data.
      Disabling the conversion webhook will break garbage collection.

  - What consequences does it have on existing, running workloads?

    Examples:
    - New namespaces won't get the finalizer "xyz" and hence might leak resource X
      when deleted.
    - SDN pod-to-pod routing will stop updating, potentially breaking pod-to-pod
      communication after some minutes.

  - What consequences does it have for newly created workloads?

    Examples:
    - New pods in namespace with Istio support will not get sidecars injected, breaking
      their networking.

- Does functionality fail gracefully and will work resume when re-enabled without risking
  consistency?

  Examples:
  - The mutating admission webhook "xyz" has FailPolicy=Ignore and hence
    will not block the creation or updates on objects when it fails. When the
    webhook comes back online, there is a controller reconciling all objects, applying
    labels that were not applied during admission webhook downtime.
  - Namespaces deletion will not delete all objects in etcd, leading to zombie
    objects when another namespace with the same name is created. -->

## Alternatives

Similar to the `Drawbacks` section the `Alternatives` section is used
to highlight and record other possible approaches to delivering the
value proposed by an enhancement, including especially information
about why the alternative was not selected.

## Infrastructure Needed [optional]

Use this section if you need things from the project. Examples include a new
subproject, repos requested, github details, and/or testing infrastructure.
