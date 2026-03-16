
# Access Management

## Main principles

You have to distinguish between:
- identity management / authentication: IDP, Cognito User Pool, AWS Identity Center Users.
- authorization: roles, policies, AWS Identity Center permission sets. For cross-account access: an IAM role is enough - users do not need an IAM user to access an AWS account when using IAM Identity Center or federated access (e.g., via IDP + SAML/OIDC)
- and access mechanism (STS tokens, Cognito Identity Pool).

| Scenario | IAM Identity Center | Cognito | STS |
| -------- | ------------------- | ------- | --- |
| **Enterprise employees accessing AWS** | ✅ Ideal (SSO to AWS accounts, MFA enforcement) | ❌ Not designed for this | ✅ Used under the hood, when IAM Identity Center calls STS to generate temporary credentials for an IAM role |
| **Developers needing access to specific AWS resources** | ✅ Yes (via permission sets) | ✅ Yes (via Identity Pools + IAM roles) | ✅ Primary solution (direct STS calls) |
| **Consumer-facing app (e.g., mobile)** | ❌ No | ✅ Ideal (user sign-up, social logins, app auth) | ✅ Used under the hood, when Cognito Identity Pools call STS to generate temporary credentials for an IAM role |
| **SSO for business apps (e.g., SAP)** | ✅ Yes | ❌ No | ❌ Irrelevant (STS is for AWS resources only) |
| **Guest/unauthenticated access** (e.g. a mobile app lets users browse public content in S3 without requiring a login) | ❌ No | ✅ Yes (via Identity Pools) | ✅ Used under the hood, when Cognito calls STS to generate temporary credentials for the unauthenticated role with restricted permissions |

Usage of specific services and flows depends on scenario.
- Do you want to allow a user to authenticate in a SaaS using an AWS SSO service? → AWS IAM Identity Center
- Does user need to access AWS resources? → AWS IAM Identity Center, STS or Cognito
- You build a consumer-facing app hosted on AWS and need authentication / authorization functionality? → Cognito
- Does your script need to programmatically access AWS resources? → STS
- Do you want to rely on AWS services or use federation? → AWS IAM Identity Center vs. IDP like AD or Okta

Flows typical for service integration:
- User → [authentication via] IDP / SSO system, protocol: SAML / OIDC → map group in vendor software based on role in IDP → to data inside vendor software
- Developer → [authentication via] IDP (Okta/Azure AD) → [Gets] JWT/OIDC token → [App/Script calls] AWS STS (AssumeRoleWithWebIdentity) → [STS returns] Temporary AWS credentials → access to S3 bucket or other resources.
- Developer → [authentication via] IDP (Okta/Azure AD) → IAM Identity Center → [Identity Center assigns] Permission Set (IAM Role) → [Developer gets] Temporary AWS credentials → access to S3 bucket or other resources.



## More details

More detailed flow:
- User:
    - Client IDP (Identity Provider): Use your enterprise SSO (e.g., Okta, Azure AD) for authentication. This provides a seamless login experience and centralizes user management. Protocol: SAML 2.0 or OIDC (OpenID Connect) for federated login.
    - Role-Based Access Control (RBAC): Assign roles (e.g., "Analyst," "Viewer") in your application, mapped to groups in your IDP.
    - AWS IAM (Optional): If users need direct access to AWS resources (e.g., S3 buckets), use IAM roles with temporary credentials (via AWS STS) and attribute-based access control (ABAC).
- Developer:
    - If developers are part of your enterprise, you can also use client IDP for initial authentication, then exchange the token for AWS credentials.
    - Then, Developer authenticates with your enterprise IDP (e.g., Okta, Azure AD) using OIDC/SAML.
    - IDP issues a token (e.g., JWT) to the developer.
    - Application exchanges the token for temporary AWS credentials using AWS STS (AssumeRoleWithWebIdentity or AssumeRoleWithSAML).
    - Developer uses AWS credentials to access AWS services (S3, Redshift, etc.) via APIs.
    - Renew access. The IDP token (JWT/SAML) has its own expiration (e.g., 1 hour). The AWS temporary credentials (from STS) also expire (default: 1 hour, max: 12 hours). To renew access: The application must detect expired credentials and re-authenticate with the IDP (if the IDP token expired) or call AWS STS again (if only AWS credentials expired). This is typically handled by the application’s SDK or a credential provider (e.g., AWS SDK auto-refreshes STS credentials if the session is still valid).

One of the complex cases from my history:
- tech user should have been added to Active Directory and get an email
- then added to IDP in 2 environments (QA and PROD)
- then added to vendor software
- and also I had to do something in Gatekeeper as reverse proxy

### AWS services
What is AWS Cognito?
- AWS Cognito can simplify the process of federating your enterprise IDP with AWS IAM, especially for developers and applications. Here’s how it fits into your scenario. AWS Cognito acts as a middle layer between your Client IDP (e.g., Okta, Azure AD) and AWS IAM. It provides: User Pools: For managing user identities and authentication (e.g., OIDC/SAML). Identity Pools: For exchanging IDP tokens (e.g., JWT) for AWS credentials via STS
- Amazon Cognito User Pools handle authentication (sign-up/sign-in, user directories, and JWT tokens), acting as an identity provider. Cognito Identity Pools handle authorization (exchanging tokens for temporary AWS credentials to access services like S3 or DynamoDB). They are often used together: User Pool verifies who the user is, and Identity Pool determines what they can access.
- ✅ Use only User Pools if your app needs to authenticate users but doesn’t require direct AWS resource access.
- ✅ Use only Identity Pools if you already have an IDP (e.g., Okta) and just need to grant AWS access.

What is AWS Identity Center (successor to AWS SSO)?
- Federates with external identity providers (e.g., Okta, Azure AD, Active Directory).
- Provides a single sign-on (SSO) portal for users to access multiple AWS accounts and applications.
- Assigns permission sets (predefined IAM roles) to users/groups for AWS access.
- Supports multi-account access (e.g., dev, staging, prod) without switching roles manually.
- When to Use IAM Identity Center? For employees or contractors who need to log in to the AWS Console or SaaS apps (e.g., Salesforce). For centralized access management across multiple AWS accounts. For enforcing SSO and MFA for human users.
- For programmatic access use STS


--------------------------------------

# Zero Trust Networks on AWS

Principles:

1. Identity and Access Management (IAM).

- MFA
- Least-privilege access
- RBAC. To avoid role explosion but keep fine-grained access control, use ABAC (Attribute-Based Access Control).
- Look above for more details.

2. Network Segmentation

- create VPC (don't use the default one)
- Isolate resources in subnets. Limit access through Security Groups and Network ACLs

3. Access policies

- Consider use-cases that require firewalls: workload-to-workload, client-to-workload, and workload-to-internet traffic flows
- use AWS WAF to protect web applications (Load balancers, API Gateway, AWS CloudFront) from common exploits, malicious bot traffic, and SQL injection/XSS attacks by filtering HTTP/S traffic based on customizable rules (e.g. to restrict IP ranges)
- granular access controls for instances and subnets via Security Groups and Network ACLs

4. Automation & centralized governance

- AWS Service Control Policies (SCPs) are JSON-based policies used in AWS Organizations to centrally manage maximum available permissions across multiple AWS accounts. They override IAM permissions.
- AWS Config: monitor resources and enforce compliance. Examples: lambda_deprecated_runtime or securitygroup_egress_allportsallow_check. More examples: [github repo](https://github.com/aws-samples/aws-config-custom-policy-rule-samples)

5. Monitoring

- CloudTrail
- AWS Security Hub
- CloudWatch
- VPC Flow Logs

More advanced topics: [link](https://aws.amazon.com/de/events/reinforce/on-demand/)
