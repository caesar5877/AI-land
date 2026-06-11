Below, I use Kafka SaaS to refer to your Confluent Cloud onboarding process.

For tomorrow’s session, keep the demo focused on one simple message:

Confluent Cloud manages the Kafka infrastructure for us.
Our team is responsible for onboarding the application, provisioning logical Kafka resources, configuring access, and validating the end-to-end connection.

Your uploaded notes support this structure. They describe the OIDC Resource, FID, C2C authentication, Control Plane roles, and the Kafka SaaS resource provisioning sequence: Namespace → Topic → Identity Pool → Client Quota → Preapproval → RBAC Role.  

1. Recommended Demo Format

Use a 30-minute KT session.

Section	Time	Purpose
Introduction	3 min	Explain the goal and scope
Kafka SaaS onboarding flow	12 min	Walk through the onboarding steps
Shared UAT isolation strategy	8 min	Explain how QA/UAT/PERF remain isolated
Validation and troubleshooting	5 min	Show what to check after provisioning
Q&A	2 min	Answer questions

Do not spend time explaining broker maintenance, disk capacity, Kafka upgrades, or cluster scaling. Those are managed by the platform team and Confluent Cloud.

⸻

2. Simple Demo Flowchart

Use this as your main slide.

flowchart TD
    A([Start]) --> B[Create IDAHO OIDC Resource ID]
    B --> C[Create FID]
    C --> D[Raise Ticket for C2C Connection]
    D --> E[Onboard Application through Swagger<br/>Provide Deployment ID, CLID and AWS Account]
    E --> F[Kafka SaaS Generates Logical Cluster]
    F --> G[Kafka Team Starts Network Provisioning<br/>Cutoff: Tuesday 3 PM<br/>Provision Day: Friday]
    G --> H{Wait for Network<br/>Before Continuing?}
    H -->|No| I[Continue Resource Onboarding in Parallel]
    I --> J[Create Namespace]
    J --> K[Create Topic Names]
    K --> L[Create Identity Pool]
    L --> M[Request Client Quota]
    M --> N[Complete Preapproval if Required]
    N --> O[Create RBAC Roles]
    H -->|Yes| P{Is Network Provisioned?}
    P -->|No| Q[Follow Up and Wait for Next Provision Cycle]
    Q --> P
    P -->|Yes| R[Validate Network Connection]
    O --> R
    R --> S[Configure Application]
    S --> T[Run End-to-End Test]
    T --> U{Publish and Consume<br/>Successful?}
    U -->|No| V[Check Network, Topic, Identity Pool,<br/>Quota and RBAC]
    V --> T
    U -->|Yes| W([Complete])

⸻

3. Isolation Strategy Slide

Use a second slide to explain why separate resources are required when multiple application environments share the same Confluent Cloud UAT platform.

flowchart TD
    A[Shared Confluent Cloud UAT Platform] --> B[Shared UAT Kafka Infrastructure]
    B --> C[QA Logical Boundary]
    B --> D[UAT Logical Boundary]
    B --> E[PERF Logical Boundary]
    C --> C1[Unique Resource ID]
    C --> C2[Unique FID]
    C --> C3[Unique Namespace and Topic Names]
    C --> C4[Unique Identity Pool]
    C --> C5[Unique RBAC and Quota]
    D --> D1[Unique Resource ID]
    D --> D2[Unique FID]
    D --> D3[Unique Namespace and Topic Names]
    D --> D4[Unique Identity Pool]
    D --> D5[Unique RBAC and Quota]
    E --> E1[Unique Resource ID]
    E --> E2[Unique FID]
    E --> E3[Unique Namespace and Topic Names]
    E --> E4[Unique Identity Pool]
    E --> E5[Unique RBAC and Quota]

Your explanation should be:

QA, UAT, and PERF share the same managed Kafka infrastructure, but they do not share the same logical resources.
We isolate them by identity, topic name, resource boundary, permissions, and quota.

⸻

4. Suggested Demo Walkthrough

Use one real example throughout the presentation. For example:

Application: accounts-events-processor
Environment: QA
Topic: qa-accounts-inbox

Then explain that UAT and PERF repeat the same pattern:

qa-accounts-inbox
uat-accounts-inbox
perf-accounts-inbox

If your team has already agreed on a naming pattern such as:

<environment>-<subdomain>-<topic-name>

show it explicitly:

qa-accounts-inbox
uat-accounts-inbox
perf-accounts-inbox

This is easier for the audience to understand than discussing every possible resource name.

⸻

5. Full Transcript

Opening

Hi everyone. Today I will walk through our Kafka SaaS onboarding process and explain how we use Confluent Cloud.

The main point is that we are not managing Kafka infrastructure ourselves. Confluent Cloud and the Kafka SaaS platform manage the Kafka cluster, brokers, availability, and infrastructure operations.

Our responsibility is to onboard the application, create the required logical resources, configure identity and permissions, and validate the end-to-end message flow.

I will also explain how we isolate multiple application environments, such as QA, UAT, and PERF, even though they share the same Confluent Cloud UAT platform.

⸻

Step 1: Create the IDAHO Resource ID

The first step is to create an IDAHO OIDC Resource ID.

This resource defines the authentication boundary for the application. It allows us to associate application roles with the Kafka SaaS identity model.

You can think of this Resource ID as the application’s identity container. It is the starting point for mapping our application identity into Kafka SaaS.

Your uploaded notes describe this OIDC Resource as the logical authentication boundary used later for Identity Pool role mapping. They also note that the combined resource and role name must stay within the Confluent naming limit.  

⸻

Step 2: Create the FID

The second step is to create the FID, which stands for Functional ID.

The FID is a non-human service identity. It is not a personal SID.

We use it for application-to-application authentication and for Control Plane operations where a service identity is required.

In simple terms, the FID is the service account representing the application.

Your uploaded notes explain that the FID is a project-level service identity and that production Control Plane operations must use an FID rather than a personal SID.  

⸻

Step 3: Raise the C2C Connection Ticket

After the Resource ID and FID are ready, we raise a ticket to establish the C2C connection.

C2C means client-to-client authentication.

This allows the application to obtain the required token and authenticate to the Kafka SaaS platform using its service identity.

⸻

Step 4: Onboard the Application through Swagger

Next, we onboard the application through the Swagger API.

During onboarding, we provide the required application information, including the Deployment ID, CLID, and AWS account information.

After the onboarding request is processed, Kafka SaaS generates the logical cluster information for the application.

At this point, clarify the boundary:

The logical cluster is the application-facing Kafka boundary.
The underlying Kafka infrastructure remains managed by the platform.

⸻

Step 5: Start Network Provisioning

After the application onboarding is submitted, the Kafka team sets up the required network connection.

Based on our current process, the cutoff is Tuesday at 3 PM and the provisioning day is Friday.

On Friday, we should verify whether the network has been provisioned successfully.

Then explain the important optimization:

We do not need to stop all work while waiting for the network provisioning.
Resource onboarding can continue in parallel.

⸻

Step 6: Provision Kafka SaaS Resources in Parallel

While network setup is in progress, we can start provisioning the required Kafka SaaS resources.

The recommended sequence is:

Namespace, Topic, Identity Pool, Client Quota, Preapproval, and RBAC Role.

Your uploaded notes explicitly list this provisioning order.  

Namespace

Namespace is the logical grouping boundary.

You can think of it as a folder that groups related topics and helps define the topic prefix.

Topic

Topic is the message channel.

The producer writes events to the topic and the consumer reads events from the topic.

Identity Pool

Identity Pool maps the external IDAHO role into the Kafka SaaS authorization model.

This is the key connection between the application identity and the Kafka resources.

Client Quota

Client Quota protects the shared cluster from excessive traffic.

This is especially important for PERF because load testing can affect other environments if we do not control the traffic.

Preapproval

Depending on the platform workflow, preapproval may be required before final access can be granted.

RBAC Role

RBAC defines what the identity is allowed to access.

For example, the application may need permission to produce messages, consume messages, or manage resources.

The uploaded notes also describe the Control Plane roles: Viewer, Operator, and Provisioner.  

⸻

6. Isolation Strategy Transcript

After explaining the onboarding steps, transition to the shared UAT architecture.

Now I want to explain the isolation strategy.

Our Confluent Cloud platform has a shared UAT environment. However, our applications may have multiple UAT-like environments, such as QA, UAT, and PERF.

These application environments share the same managed Kafka infrastructure, but they must not share the same logical Kafka resources.

Then show the resource table:

Resource	QA Example	UAT Example	PERF Example
Resource ID	accounts-qa-resource	accounts-uat-resource	accounts-perf-resource
FID	accounts-qa-fid	accounts-uat-fid	accounts-perf-fid
Namespace	qa-accounts	uat-accounts	perf-accounts
Topic	qa-accounts-inbox	uat-accounts-inbox	perf-accounts-inbox
Identity Pool	qa-accounts-pool	uat-accounts-pool	perf-accounts-pool
RBAC	Access to qa-* only	Access to uat-* only	Access to perf-* only
Quota	QA traffic limit	UAT traffic limit	PERF traffic limit

Continue:

We isolate the environments in several layers.

First, each environment has its own Resource ID and FID, so the application identity is separated.

Second, each environment has its own topic name. We use the environment as part of the topic naming convention.

For example:

qa-accounts-inbox
uat-accounts-inbox
perf-accounts-inbox

Third, each environment has its own Identity Pool.

Finally, RBAC ensures that a QA identity can only access QA resources, a UAT identity can only access UAT resources, and a PERF identity can only access PERF resources.

This gives us logical isolation even though the underlying Confluent Cloud UAT infrastructure is shared.

Use this analogy:

Think of the shared Confluent Cloud UAT cluster as one office building.

QA, UAT, and PERF are three separate teams working inside that building.

Each team has its own badge, its own rooms, its own files, and its own access rules.

They share the building, but they should not be able to open each other’s doors.

⸻

7. Validation Demo Transcript

After provisioning and network setup:

Once the resources and network connection are ready, we configure the application and run an end-to-end test.

The producer should publish an event to the environment-specific topic.

The consumer should read the event from the same topic.

We should also verify that the consumer group is environment-specific and that the correct RBAC permissions are applied.

Show a simple flow:

QA Producer
  ↓
qa-accounts-inbox
  ↓
QA Consumer

Then explain the failure path:

If the producer cannot publish or the consumer cannot read, we should check the following items in order:

Network status, topic name, identity mapping, Identity Pool, RBAC permissions, and Client Quota.

⸻

8. Troubleshooting Slide

Use this concise table.

Symptom	First Checks
Application cannot connect	Network status, bootstrap configuration, C2C token
Producer cannot publish	Topic name, Identity Pool, producer RBAC
Consumer cannot read	Topic name, consumer group, consumer RBAC
Requests are throttled	Client Quota and throttling logs
Provisioning is blocked	Preapproval, networking status, Control Plane permissions
QA messages appear in UAT	Topic naming, Resource ID, Identity Pool, RBAC scope

⸻

9. Closing Transcript

To summarize, Confluent Cloud manages Kafka infrastructure for us.

Our team is responsible for four main areas:

First, create the application identity using the Resource ID and FID.

Second, onboard the application and establish the C2C and network connection.

Third, provision the Kafka SaaS resources: Namespace, Topic, Identity Pool, Client Quota, Preapproval, and RBAC.

Fourth, validate the producer and consumer flow end to end.

Because QA, UAT, and PERF share the same Confluent Cloud UAT platform, we use unique identities, topic names, Identity Pools, quotas, and RBAC permissions to isolate them logically.

The key takeaway is that the infrastructure is shared, but the application resources and permissions must remain isolated.

⸻

10. Questions to Expect

Prepare these answers before the session.

Why do we need a unique topic name for every environment?

Because QA, UAT, and PERF share the same Confluent Cloud UAT infrastructure. Unique topic names prevent message traffic from mixing across environments.

Why do we need separate Identity Pools if topic names are already different?

Topic names separate the message channels. Identity Pools and RBAC enforce the access boundary. They prevent an application from accidentally reading or writing another environment’s topic.

Can resource onboarding start before the network is ready?

Yes. Network provisioning and Kafka resource onboarding can proceed in parallel. The network must be ready before the final end-to-end validation.

Why is Client Quota important?

Because the cluster is shared. Client Quota prevents one workload, especially PERF testing, from consuming too much shared capacity.

What does the platform team manage?

The platform team manages the Kafka infrastructure, logical cluster provisioning, and network setup. The application team manages application onboarding, resource requests, configuration, and end-to-end validation.

⸻

11. Final Preparation Checklist

Before the KT session, prepare:

[ ] One onboarding flowchart
[ ] One isolation strategy diagram
[ ] One real QA example
[ ] One table comparing QA / UAT / PERF resources
[ ] One Swagger onboarding screenshot
[ ] One networking status screenshot
[ ] One example topic and Identity Pool
[ ] One RBAC role example
[ ] One troubleshooting slide
[ ] One backup screenshot in case the live system is unavailable

During the demo, use screenshots for destructive or slow provisioning steps. Use the live system only to show an existing logical cluster, resource status, topic, Identity Pool, and RBAC configuration.