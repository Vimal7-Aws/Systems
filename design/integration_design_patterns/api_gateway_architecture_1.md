![img.png](img.png)


Multi-region API Gateway with CloudFront

In this architecture, we deploy API Gateway in multiple regions and use CloudFront as a global CDN to route requests to the nearest API Gateway instance. This setup provides low latency and high availability for global users.




Deploy an API endpoint in two or more AWS Regions using Amazon API Gateway, then handle requests using AWS Lambda connected to an Amazon Aurora relational database.

To keep data in sync across all AWS Regions, enable the Aurora Global Database feature. Aurora automatically routes write requests to the primary node to support data transactions, and replicates the changes to all nodes across AWS Regions.

To support custom domains, upload the domain’s SSL Certificate into AWS Certificate Manager (ACM) and attach it to an Amazon CloudFront distribution.

Point your domain name to CloudFront by using Amazon Route 53 as your DNS name resolution service.

Set up a routing rule on Route 53 to route your global clients to the AWS Region with least latency to their location.

Ensure clients can authenticate seamlessly to API Gateway endpoints in any Region by using AWS Lambda @Edge to query Route 53 for the best AWS Region to forward the request to. This normalizes authorization by abstracting the specifics of each regional endpoint.

Clients across the globe can then connect to your APIs using a single endpoint available in edge locations.

CloudFront will seamlessly route client requests from edge locations to the API in the AWS Region with the lowest latency to the client’s location.
