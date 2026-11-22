CloudMed Solutions: Zero Trust Azure Landing Zone

System: MedConnect
Regions: Canada Central and West Europe
Scope: this document is entirely about the conceptual architecture and governance design. It has absolutely nothing to do with the actual deployment of any resources.

1. Company Overview

CloudMed Solutions Inc. provide telemedicine and patient management services to hospitals and clinics across Canada, the US and Europe. Their MedConnect platform facilitates doctor to patient consultations, EMR management and AI driven health analytics. Web and mobile apps access their encrypted Azure SQL databases and Azure Storage via backend APIs.

Patient records, clinical communications and analytics data are all extremely sensitive. A breach of an account, application or storage endpoint could have a huge impact on patient privacy and on service availability. CloudMed validates every access request, enforces least privilege on every identity and uses segmentation to contain any incidents. Network location in itself does not confer access.

The proposed landing zone establishes consistent subscription boundaries, shared security services and centrally managed guardrails which is of course in line with the platform and workload separation described in Microsoft's Azure landing zone overview.

Compliance and operational drivers

Column 1 Column 2
Driver CloudMed design response
HIPAA: protect ePHI with access controls, auditing, encryption, incident response and appropriate Business Associate agreements. Make sure the services and contracts you've selected are in scope.
GDPR: Document processing purposes and lawful bases, minimize personal data, support data subject requests and assess international transfers. Keep the EU clinical datasets in the approved EU deployment by default.
PIPEDA: Assign privacy accountability, establish consent and retention processes, protect the information with appropriate safeguards and assess any third party processing arrangements.
Availability and growth: use repeatable regional deployments, supported availability zone configurations, tested backups and independent workload subscriptions.
Central operations: Standardize policy, identity, monitoring, incident response and cost allocation across both regions.

These controls will help you to be compliant but architecture on its own won’t get you certified. CloudMed is still ultimately responsible for the organization’s processes, contracts and evidence. See Microsoft’s HIPAA guidance, Microsoft’s GDPR guidance and the Canadian privacy regulator’s PIPEDA accountability guidance.

Regional assumptions

* Canadian clinical data is stored in Canada Central and EU clinical data is stored in West Europe (with regional backups and sensitive app logs). That's CloudMed's proposed residency policy and it doesn't in any way mean that the local laws require the data to be stored in those locations.
* US customers could only use Canada Central once the contractual and privacy reviews had approved it. If US-only hosting was required then the existing two region constraint would have to be changed.
* The two deployments support separate regional datasets. Patient records are not automatically replicated or failed over between the regions.
* Use in region redundancy wherever possible. A full regional outage is still a recovery limitation under the two region and residency rules. You'll need approval of an additional recovery region in the same geography prior to you being able to commit to regional DR.
* Review the service specific residency, the identity metadata, the telemetry destinations and any telehealth media relay individually. Resource location policies in themselves don't provide end to end residency.
1. Governance and Identity

Management group and subscription hierarchy

Tenant Root Group
└── CloudMed                         [management group: common guardrails]
├── Platform                     [management group]
│   ├── Connectivity-CA          [subscription: Canada Central hub]
│   ├── Connectivity-EU          [subscription: West Europe hub]
│   └── Management-Security      [subscription: regional monitoring]
├── Production                   [management group: strict controls]
│   ├── MedConnect-Prod-CA        [subscription: App/API/Data spokes]
│   └── MedConnect-Prod-EU        [subscription: App/API/Data spokes]
└── Development                  [management group: synthetic data only]
├── MedConnect-Dev-CA        [subscription]
└── MedConnect-Dev-EU        [subscription]

Minimize tenant root assignments. Apply organisation wide policies at the CloudMed level and more restrictive policies at Production. Within each workload subscription split the app, API, data and networking resources into separate resource groups. Regional subscription policies will restrict allowed locations to that subscription’s region. Dev has no network path and no data permissions to Production.

RBAC model

Assign Azure roles to Microsoft Entra security groups rather than to individual users. Keep the Azure management plane permissions separate from the database and storage data plane permissions. Microsoft recommend that you assign the appropriate landing zone roles at the correct scope. See identity and access management for landing zones.

Column 1 Column 2 Column 3
Team or identity Proposed access and scope Restrictions
Platform Administrators: Eligible Network Contributor on connectivity resources; eligible Contributor on assigned platform resources. To be activated via PIM with approval, MFA, justification and expiry. Separate role assignment administration.
Identity/access administrators: Eligible Entra administrative roles and Azure RBAC Administrator only at the necessary scopes. Entra roles and Azure RBAC roles are separate. Review elevation and assignments. Avoid standing Owner access.
DevOps engineers: Contributor to the development application resource groups; Reader in Production. Production writes using the approved pipelines and with very narrowly scoped deployment identities. Human production elevation is extremely rare and time limited.
Finance Cost Management Reader at subscriptions; Cost Management Contributor only for budget maintainers. No clinical data access, no secret access, no resource deployment, no IAM administration. Billing scope access must be validated separately.
Security operations: Security Reader and scoped log access; eligible incident response roles. Restrict clinical log access and record privileged investigations.
Workloads: Separate managed identities. The API receives only the required SQL permissions and storage data roles. The App has no direct database access. No shared admin credentials.

Identity security: use Microsoft Entra ID for workforce authentication. Conditional Access requires MFA, compliant managed devices for privileged work and approved access conditions (location and risk where supported). Phishing resistant authentication is required for admins, legacy auth is blocked and guest access is regularly reviewed. Emergency access accounts are monitored with very narrow exceptions and recovery tests are performed on a periodic basis.

Patients authenticate via a separate customer identity configuration (e.g. Microsoft Entra External ID) and their personal devices are not regarded as corporate compliant. MedConnect carries out the token issuer, audience, expiry, scopes and hospital/patient authorisation checks on every API call. Azure RBAC in no way replaces the application level patient access checks.

Managed identities and federated pipeline identities don't require stored credentials. Human MFA and device rules don't come into play for managed identities. You secure them via resource permissions, credentialless auth, network boundaries and monitoring.

Azure Policy guardrails

Column 1 Column 2
Policy area Proposed enforcement
Required metadata: Owner, Environment, Cost Centre, Application and Data Classification. Inherit the approved tags via Modify and Remediation where possible. Never store patient information in tags.
Allowed regions: Deny all regional resources outside of Canada (Central and West Europe); narrow each workload subscription down further. Global services as required will be handled via scoped exceptions.
Public exposure: Deny public IPs on workload VMs and on any unapproved resources. Disable public network access on SQL, Storage and Key Vault via service specific policies.
Approved resource types/SKUs, naming standards as defined through the deployment modules, required subnet NSGs, encryption settings and secure transport. Custom policies to be used only where the built-ins and aliases do not meet the requirement.
Visibility Audit or deploy the supported diagnostic settings and security configurations using DeployIfNotExists plus a remediation identity.

Test your policies in Dev prior to enforcing them in Production. A deny will stop the non-compliant change from being applied but won't actually fix the resources that are non-compliant; the audit findings need to be remediated. For each exemption you have you need to have the owner, the scope, the compensating controls and the expiry all documented. You need to be very clear about what you're permitting in terms of the approved WAF ingress and the required platform public endpoints rather than just exempting the entire subscription.

1. Network Architecture

Regional hub-and-spoke topology

Deploy one connectivity hub per region with separate Production App, API and Data spoke VNets. Non-overlapping address spaces are required (e.g. 10.10.0.0/16 for Canada Production, 10.20.0.0/16 for Europe Production) and those should be broken down into hub and spoke ranges. Allocate distinct ranges for Dev and for the future.

Each spoke only peers out to its own regional hub. Peering is not transitive. Spoke to spoke flows that are approved need to be explicitly routed via Azure Firewall, have the appropriate forwarding in place and have a symmetric return path. Hub membership does not imply trust. The design is consistent with Microsoft's hub and spoke and network segmentation guidance.

Column 1 Column 2
Component Location and purpose
Azure Firewall Premium (Dedicated) AzureFirewallSubnet in each hub; default deny inter-spoke and outbound rules, threat intelligence and IDS/IPS. TLS inspection to be evaluated on a selective basis with due regard to privacy and certificate requirements.
Azure Bastion (Dedicated) AzureBastionSubnet; provides the means for controlled administrative access to private VM addresses without the need for public SSH/RDP. Where possible, prefer a private only deployment and access via the approved private connectivity.
DNS: Azure DNS Private Resolver in dedicated hub subnet, with centrally managed Private DNS zones linked to the relevant VNets. Private endpoint names resolve privately.
Private connectivity via VPN or ExpressRoute as required for hospital/private administration paths. Routes and permissions remain restricted.
Log Analytics regional workspaces in the Management-Security subscription are logically shared with each hub. Log Analytics is a managed service (it's not a server or a subnet within the hub VNet).

Workload isolation and permitted flows

The App spoke has a dedicated Application Gateway WAF subnet and a private application compute subnet. The API spoke has private backend compute and an isolated analytics subnet. The Data spoke has private endpoint subnets for SQL, Storage and Key Vault. Azure SQL and Storage are still managed services outside of the VNet, accessed via private endpoints.

Column 1 Column 2 Column 3
Source → destination Allowed flow Enforcement
Patients/clinicians → regional WAF → App HTTPS 443 WAF prevention rules, TLS, rate controls, application authentication and authorisation; only approved ingress is public.
App → API via HTTPS 443 through regional firewall (explicit firewall allow rule, subnet NSGs, validated tokens and API authorisation).
API → SQL private endpoint (SQL TLS, TCP 1433) using a selected Proxy connection policy. API only source, endpoint subnet NSG, database permissions, managed identity. Reassess ports if moving to Redirect API → Storage/Key Vault private endpoints (HTTPS 443) Explicit routes and firewall rules, endpoint NSGs and narrowly scoped data permissions.sAnalytics → approved data. Required private service ports only. Dedicated identity. Approved views/datasets. Minimisation. No unrestricted raw record export.xAdmins → Bastion → approved VMs. Required SSH/RDP management ports. PIM activation, Conditional Access, scoped VM permissions and logged sessions where applicable.oWorkloads → external dependencies: Approved destinations via firewall. Explicit outbound allowlists. No unrestricted internet egress.grSubnet NSGs will block all unwanted lateral traffic (including the default VNet access which would effectively break the isolation). Don't route lateral traffic between subnets through the firewall by default. Use subnet separation and host/application level controls. East-west traffic is not allowed in Dev or Production.ctEnable private endpoint network policies on the NSGs and on the routes for the endpoint subnets. Use specific routes to ensure that the private endpoint system routes go through your inspection (don't just rely on a default route to prove that private link traffic goes through your firewall). Validate both the forward and the return path. See private access to Azure PaaS services.viBe absolutely clear that public network access is to be turned off for both SQL and Storage (just creating a private endpoint isn't enough). Use the standard service hostnames in your app connection config and resolve those via Private DNS. See the Azure SQL Private Link guidance.daTelehealth signaling uses the authenticated application or API path. Real time media requires a separate validated media or relay design, residency assessment and very narrowly approved ports. The HTTPS application path will not permit all the audio and video traffic through Application Gateway.teway.

1. Zero Trust Controls

The architecture implements Microsoft’s three Zero Trust principles for landing zones:

Column 1 Column 2
Principle CloudMed implementVerify explicitly: validate user and workload identities, apply workforce MFA and Conditional Access, authenticate API calls and verify patient/tenant authorisation for every protected operation.ation.
Use least privilege, Scope RBAC and data permissions as narrowly as possible. Activate admin roles via PIM and restrict deployment identities. Periodically review access.
Assume breach. Isolate environments and tiers, filter traffic, encrypt data, retain protected audit evidence and rehearse containment and recovery.

Five specific design examples:

1. Bastion with just in time administration: an engineer takes on a PIM role for a limited period of time prior to accessing an approved private VM. Where applicable Defender for Cloud JIT VM access will further restrict the exposure of the management port (this is in addition to the PIM role activation).
2. Private Link for SQL: Only the authorized API identity can query the required database objects over the approved private endpoint. Application servers cannot connect directly and public access to SQL is disabled.
3. Policy is that any VM deployment which requests a public IP is blocked. There is one very specific, documented exception which allows the regional WAF's approved ingress endpoint.
4. Encryption and secret isolation: TLS is required for application and data connections, SQL and Storage are encrypted at rest. Secrets that need to be stored are in Key Vault with rotation and recovery protection enabled. Customer managed keys are used where required by the threat model or contracts and where operationally viable.
5. Containment in the event of suspected compromise: isolate the affected compute, disable or restrict the compromised identities, preserve the audit trail, investigate access to patient records, rotate the exposed secrets and restore from a verified backup. Privacy will assess their notification obligations.

Use synthetic data in development. Limit analytics to its intended purpose and never include patient records in any prompts or model training datasets without explicit permission. Patient data should only be retained as per the approved retention schedule. Test backup restore and application authorisation including attempts to access records from other hospitals.

1. Monitoring, Compliance, and Cost

Central visibility with regional data boundaries

Collect Azure Activity Logs, resource diagnostics, Entra audit and sign-in events, WAF and firewall events, database audits and application telemetry as appropriate. Monitor availability, latency, errors and suspicious activity via regional Log Analytics workspaces and Application Insights. Global security dashboards must be querying into approved regional data only (no consolidation of all the clinical logs into a single region). Global identity logging will be reviewed separately. Azure Monitor is the observability foundation.

Enable the correct Defender for Cloud plans for the relevant compute, SQL and Storage services. Use Microsoft Sentinel for correlated investigation and response as appropriate. Set up alerts for privilege changes, multiple failed authentications, unusual record access, suspicious exports, policy changes, security service disablements and service outages. Each alert must have an owner, severity and a response runbook.

Don't include tokens, secrets and clinical payloads in the standard telemetry. Log access must be restricted and audit archives must be protected against tampering. Hot log and archive retention periods should be in line with the clinical, legal and security requirements as documented rather than applying a single retention period to all regulations.

Compliance enforcement and evidence

Use the Azure Policy compliance views and Defender for Cloud's regulatory compliance dashboard to track both the technical controls and the remediation. Assign the relevant standards to each service and subscription and map any unmet PIPEDA or contractual obligations to custom controls and manual evidence. A green dashboard in no way constitutes legal certification. Platform owners do weekly drift reviews. Security and privacy teams do their monthly reviews of the exceptions, access, incident exercises, backup tests and compliance evidence. The control register will contain the requirements, the technical configuration, the accountable owner, the test results and the location of the evidence. Manual evidence will be maintained for consent, agreements, training and data subject request handling.

Costs: Establish monthly budgets for subscriptions and for workload/resource groups. Alert owners and Finance at actual spend thresholds of 50%, 80% and 100% as well as at the forecasted over budget threshold.

* AppApply the correct tags for allocation by application, environment, owner and cost center. Look at any untaggable or shared service costs and apply a documented allocation rule.onitor firewall processing, log ingestion and retention, private endpoints and data transfer costs as well as your compute and databases.
* RigGet the underused resources right, plan the development shutdowns and review the savings commitments once usage is predictable. Make sure clinical capacity and recovery are protected as we go through the optimization.reat budget alerts as notifications rather than as hard spending caps. Any automated action must be specifically engineered. Production clinical services will not simply be stopped if the budget is exceeded. See Azure budget behavior.
1. Conceptual Diagram

GitGitHub renders the following Mermaid diagram directly in this README. Solid arrows represent hierarchy or labeled traffic flows. Dashed arrows represent governance, identity, monitoring or implementation relationships. The repeated regional boxes are equivalent deployments (they're not indicative of cross regional replication paths).lowchart TB
ROOT[“Tenant Root Group”] --> CM[“CloudMed management group”]
CM --> PM[“Platform management group”]
CM --> PR[“Production management group”]
CM --> DV[“Development management group”]
PM --> CON[“Connectivity-CA and Connectivity-EU subscriptions”]
PM --> MS[“Management-Security subscription”]
PR --> PC[“MedConnect-Prod-CA subscription”]
PR --> PE[“MedConnect-Prod-EU subscription”]
DV --> DEV[“Dev-CA and Dev-EU subscriptionsSynthetic data; isolated from production”]

GOV["Governance layer<br/>Policy, tags, regional restrictions, budgets<br/>Least privilege: scoped RBAC and PIM"] -.-> CM
ID["Identity layer<br/>Entra ID: MFA and Conditional Access<br/>Customer identity and API authorization<br/>Verify explicitly"] -.-> PC
ID -.-> PE
MON["Monitoring layer<br/>Regional Log Analytics and Azure Monitor<br/>Defender for Cloud and Sentinel<br/>Assume breach: detection and response"]
MS --> MON

subgraph CA["Canada Central - regional deployment"]
    HC["Hub VNet<br/>Azure Firewall, Bastion, DNS Private Resolver"]
    WC["App spoke VNet<br/>WAF subnet and private App subnet"]
    AC["API spoke VNet<br/>Private API and analytics subnets"]
    DC["Data spoke VNet<br/>SQL, Storage, Key Vault private endpoints"]
    SC["Managed SQL, Storage, Key Vault<br/>Outside VNet; public access disabled<br/>Encryption and minimal data permissions"]
    WC -->|"HTTPS via hub firewall"| AC
    AC -->|"Approved ports via hub firewall"| DC
    DC -->|"Private Link"| SC
    HC ---|"Peering and controlled routing"| WC
    HC ---|"Peering and controlled routing"| AC
    HC ---|"Peering and controlled routing"| DC
end

subgraph EU["West Europe - regional deployment"]
    HE["Hub VNet<br/>Azure Firewall, Bastion, DNS Private Resolver"]
    WE["App spoke VNet<br/>WAF subnet and private App subnet"]
    AE["API spoke VNet<br/>Private API and analytics subnets"]
    DE["Data spoke VNet<br/>SQL, Storage, Key Vault private endpoints"]
    SE["Managed SQL, Storage, Key Vault<br/>Outside VNet; public access disabled<br/>Encryption and minimal data permissions"]
    WE -->|"HTTPS via hub firewall"| AE
    AE -->|"Approved ports via hub firewall"| DE
    DE -->|"Private Link"| SE
    HE ---|"Peering and controlled routing"| WE
    HE ---|"Peering and controlled routing"| AE
    HE ---|"Peering and controlled routing"| DE
end

CON -.-> HC
CON -.-> HE
PC -.-> WC
PC -.-> AC
PC -.-> DC
PE -.-> WE
PE -.-> AE
PE -.-> DE
USERS["Patients and clinicians<br/>Authenticated regional HTTPS entry"] --> WC
USERS --> WE
HC -.-> MON
HE -.-> MON
SC -.-> MON
SE -.-> MON
ZT[Assume breach: tier separation, NSGs, deny by default. Least privilege: managed identities and restricted data access. No automatic cross-region patient data replication.  -> DCT -> DEhe AppThe App-to-API and API-to-Data arrows represent the flows through the regional firewall (they don’t represent spoke peering). All tiers do export the approved telemetry, even if the individual monitoring arrows are omitted for clarity. Development uses a separate network boundary with no production route. The full flow table is in Section 3.mmary and Recommendations

CloCloudMed's landing zone is a combination of centrally governed subscriptions and independently bounded regional workloads. Identity validation protects users and services, scoped permissions limit the blast radius of any compromised credentials and private endpoints, tiering and traffic inspection all go a long way to reduce lateral movement and exposure. Regional monitoring and compliance evidence provides accountability and budgets and reusable patterns support predictable growth.he design supports compliance with HIPAA, GDPR, and PIPEDA through technical controls and clear operational ownership. Legal and contractual validation, service availability checks, routing tests, and workload-specific authorization and recovery testing are still required before implementation.

TwoTwo recommendations for the future: automate the creation of repeatable landing zones and validation. Build up your reviewed Bicep or Terraform modules and policy as code pipelines for subscription provisioning, role assignments, regional networking and monitoring. Implement checks to ensure there's no accidental public exposure, no route bypasses and no excessive data permissions before any production changes.trengthen regional disaster recovery. Set recovery time and recovery point objectives with the hospitals, assess additional recovery regions within the approved geographies and practice recovery from the immutable and protected backups. That’s the current limitation that Canada Central and West Europe can’t be used as a substitute for one another under the proposed residency rules.