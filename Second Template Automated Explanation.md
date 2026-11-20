# Automated DailyKart CloudFormation template

This guide explains [Second Template Automated.yaml](./Second%20Template%20Automated.yaml). It creates the AWS infrastructure and deploys the DailyKart app from the configured Git commit.

## What it creates

- A new VPC, two public subnets, and two private database subnets.
- An internet-facing Application Load Balancer (ALB) and an EC2 server for the app.
- A private, encrypted MySQL database. The database password is generated and stored in AWS Secrets Manager.
- A Route 53 public hosted zone and DNS aliases for the root domain and `www`.
- CloudWatch logging and an EC2 instance role for Systems Manager and reading the database secret.
- An ACM certificate and HTTPS listener after HTTPS is enabled.

The EC2 startup process installs Node.js, downloads the app, creates and seeds its database, builds the app, and starts it as a service. CloudFormation waits for the app health check before completing the first deployment.

## Deploy from the AWS Console in two phases

The domain must use the Route 53 hosted zone's name servers before ACM can validate the certificate. The template creates that hosted zone, but only the domain registrar can delegate your domain to its name servers. If HTTPS were requested in the first deployment, CloudFormation could wait for certificate validation before showing the stack outputs containing those name servers.

### Phase 1: create the infrastructure and hosted zone

1. In the AWS Console, select **CloudFormation** and make sure the console region is **Asia Pacific (Mumbai) `ap-south-1`**.
2. Choose **Create stack** → **With new resources (standard)**. Upload `Second Template Automated.yaml`.
3. Enter a stack name. On the parameters page, keep `DomainName` as `dailykarrt.online`, keep the default `AppCommit`, and set `EnableHTTPS` to `false`.
4. Continue through the remaining pages. On the review page, acknowledge that the template creates IAM resources, then submit the stack.
5. Wait until the stack status is **CREATE_COMPLETE**. Open the stack's **Outputs** tab and copy the four name servers in `Route53NameServers`.
6. At your domain registrar, replace the domain's current name servers with those four Route 53 name servers. The template creates the Route 53 zone; changing nameservers at the registrar is the one step it cannot do for you.
7. Before replacing nameservers, copy any existing DNS records you still need (for example, email MX, SPF, or DKIM records) into the new Route 53 hosted zone. Changing nameservers changes which DNS zone is authoritative for the domain.

The app and DNS records are created during phase one, and the ALB initially serves HTTP. The hosted zone exists before stack creation finishes, but waiting for `CREATE_COMPLETE` is the simplest way to get the nameservers from the Outputs tab.

### Phase 2: turn on HTTPS

1. Wait until the new nameserver delegation is visible in public DNS.
2. In CloudFormation, select the same stack and choose **Update**. Use the existing template.
3. Keep `DomainName` and `AppCommit` unchanged, and change `EnableHTTPS` to `true`. Continue through the remaining pages and submit the update.
4. Wait for **UPDATE_COMPLETE**. ACM will create its DNS validation records and issue the certificate automatically; the update then creates the HTTPS listener and changes HTTP to redirect to HTTPS. The `WebsiteURL` and `WwwWebsiteURL` outputs will now use HTTPS.

You do not manually create the certificate or its validation records, and ACM does not require a separate approval when DNS validation succeeds. It cannot issue a certificate until the domain is publicly delegated to the hosted zone and you control that DNS.

## Domain and DNS

The confirmed `DomainName` is `dailykarrt.online`. Public DNS currently reports `helios.dns-parking.com` and `aster.dns-parking.com` for that name, so it is not yet delegated to the Route 53 zone this template creates. Update the registrar nameservers only after phase one creates the hosted zone.

The `WebsiteURL` and `WwwWebsiteURL` outputs use HTTP during phase one and HTTPS after phase two. Both names are included in the certificate and point to the ALB.

## Other notes

- The stack creates billable AWS resources, including an EC2 instance, ALB, RDS database, and Route 53 hosted zone. Review AWS pricing before deploying.
- The database credentials are generated; the stack outputs the Secrets Manager ARN, not the password.
- If the app repository requires additional credentials or integrations in the future, those will need to be supplied separately; this template currently configures its MySQL environment.
