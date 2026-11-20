# Simple explanation of the second template

The [Second Template](./Second%20Template) describes a small website setup on AWS. It creates the web server and database, then places a load balancer in front of the server so visitors can use HTTPS.

## What happens when someone visits the website

1. A visitor opens `https://dailykarrt.online`.
2. Route 53 looks up the domain and points it to the Application Load Balancer (ALB).
3. The ALB presents the ACM certificate to make the HTTPS connection secure.
4. The ALB sends the web request to the EC2 server over HTTP inside the AWS network.
5. The EC2 server can connect to the private MySQL database.

The ALB also redirects visitors who use HTTP to HTTPS. The EC2 server is not open to web traffic directly; its HTTP rule accepts traffic from the ALB's security group.

## What the template creates

- **Networking rules:** Security groups allow public web traffic to the ALB, allow the ALB to reach the EC2 server, limit SSH to the IP address you provide, and allow the database to be reached from EC2.
- **EC2 web server:** A virtual machine in the public subnet. Its startup script installs updates and the CloudWatch agent.
- **Application Load Balancer:** A public entry point spread across two subnets. It forwards requests to EC2 and redirects HTTP to HTTPS.
- **ACM certificate:** A certificate request for the value of `DomainName`. DNS validation proves that you control that domain.
- **Route 53 record:** An alias record points the domain to the ALB.
- **RDS MySQL database:** A private database in the two database subnets.
- **S3 bucket:** Private object storage with encryption and versioning enabled.
- **Cognito and SES:** A user pool for application sign-in and an SES email identity.
- **IAM and CloudWatch:** An EC2 role grants the server access to the configured S3 bucket, SES email sending, and CloudWatch logging.

## Important settings to provide

When creating the stack, AWS asks for values such as the VPC, EC2 and ALB subnets, database subnets, key pair, SSH IP address, database credentials, and Route 53 hosted-zone ID.

The current default domain is **`dailykarrt.online`**. Confirm that this is exactly your registered domain. The `HostedZoneId` must refer to its **public** Route 53 hosted zone, and the domain's name servers at the registrar must point to that zone. If either is wrong, ACM may not be able to verify the domain.

The certificate and ALB must be created in the same AWS Region. The certificate and DNS record in this template cover only the apex name `dailykarrt.online`; they do not automatically cover `www.dailykarrt.online`.

## Why certificate validation can appear stuck

DNS validation is like ACM asking you to publish a special proof record for your domain. The template gives ACM a hosted-zone ID so it can arrange the validation record in Route 53. ACM still needs to see that record in public DNS before it marks the certificate as issued.

Check these items if validation remains pending:

1. The spelling in `DomainName` is the domain you actually own.
2. `HostedZoneId` is the public zone for that exact domain, not a different or private zone.
3. The registrar delegates the domain to the name servers shown for that Route 53 hosted zone.
4. In ACM, the certificate is requested in the same Region as the ALB. Look at its status and the DNS validation record it expects.
5. If the record is not visible in public DNS, check whether the expected validation CNAME was created in the correct hosted zone.

The HTTPS listener uses the certificate, so it cannot become ready until ACM issues that certificate. A stack can therefore wait at the certificate/listener stage while DNS validation is incomplete.

## Could something stop the website from working?

I don't see an obvious CloudFormation setting in the certificate section that is inherently wrong. Whether deployment succeeds still depends on the values and AWS resources you provide:

- **Certificate stuck on validation:** Make sure `dailykarrt.online` is the exact domain you own and that `HostedZoneId` is its public Route 53 zone. The domain also needs to use that zone's name servers. If any of these do not match, ACM may not be able to confirm domain ownership.
- **Website unavailable after the certificate is issued:** The EC2 startup script installs updates and the CloudWatch agent, but it does not install or start your website application. The app must already be on the server or be deployed separately. The ALB checks the server at `/`; the app must respond there for the target to be considered healthy.
- **ALB not reachable from the internet:** The two ALB subnets you provide must be public, in different Availability Zones, and have a route to an Internet Gateway.

These are possible blockers, not proof that one of them is the cause of your current issue. For an ACM validation problem, first check the domain spelling, hosted-zone ID, and domain name-server delegation.

## How this differs from the first template

The first template exposes the EC2 server directly for HTTP and HTTPS. The second template adds the ALB, DNS record, certificate, and HTTP-to-HTTPS redirect. In the second design, HTTPS ends at the ALB; the ALB communicates with EC2 over HTTP within the VPC.

## Outputs

After a successful deployment, the stack outputs items such as the website URL, ALB DNS name, EC2 address, database endpoint, S3 bucket name, and ACM certificate ARN.
