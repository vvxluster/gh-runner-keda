##How KEDA works

KEDA monitors external event sources and adjusts your app’s resources based on the demand. Its main components work together to make this possible:

KEDA Operator: keeps track of event sources and changes the number of app instances up or down, depending on the demand.
Metrics Server: provides external metrics to Kubernetes’ HPA so it can make scaling decisions.
Scalers: connect to event sources like message queues or databases, pulling data on current usage or load.
Custom Resource Definitions (CRDs): define how your apps should scale based on triggers like queue length or API request rates.

KEDA Custom Resources (CRDs)
KEDA uses Custom Resource Definitions (CRDs) to manage scaling behavior:

ScaledObject: Links your app (like a Deployment or StatefulSet) to an external event source, defining how scaling works.
ScaledJob: Handles batch processing tasks by scaling Jobs based on external metrics.
TriggerAuthentication: Provides secure ways to access event sources, supporting methods like environment variables or cloud-specific credentials.

KEDA goes beyond CPU,Ram utilization to spin up the POD, for example message queues,API,database request
