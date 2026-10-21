# ECS Fargate Service Behind an ALB (CloudFormation)

CloudFormation template that deploys an ECS Fargate cluster running one containerized service (Nginx by default) behind an internet-facing HTTPS Application Load Balancer, with CPU target-tracking auto scaling. Parameter labels in the template are in Spanish.

> **Status:** legacy reference template (2026), not actively maintained. Review and update before deploying.

## Architecture

Template: `fargate-demo.yml`

```
Internet --HTTPS:443--> ALB (ACM cert) --> target group (ip) --> Fargate tasks (private subnets)
                                                             \-> CloudWatch Logs /ecs/<ServiceName>-taskdefinition
```

- ECS cluster `<ServiceName>-ecs-cluster`
- Fargate task definition (CPU/memory chosen from mappings) with a single container and `awslogs` logging
- Execution role (`AmazonECSTaskExecutionRolePolicy` + ECR pull and log permissions), empty task role
- ECS service (2 tasks, `AssignPublicIp: DISABLED`) registered in an `ip` target group
- Internet-facing ALB with one HTTPS listener (configurable port and SSL policy)
- Security groups: ALB open on `LoadBalancerPort` from anywhere; containers reachable only from the ALB
- Application Auto Scaling target (min/max tasks) and a CPU target-tracking policy
- CloudWatch log group for the task

The tasks run in private subnets, so the VPC needs a NAT gateway or ECR/S3/Logs VPC endpoints to pull the image.

## Parameters

| Name | Purpose | Default |
|------|---------|---------|
| `VpcIdParameter` | Existing VPC | (required) |
| `ContainerSubnets` | Private subnets for the tasks | (required) |
| `LoadbalancerSubnets` | Public subnets for the ALB | (required) |
| `LoadBalancerCertificateArn` | ACM certificate in the same region | `<CERTIFICATE_ARN>` |
| `LoadbalancerSslPolicy` | ALB SSL policy | `ELBSecurityPolicy-FS-1-2-Res-2019-08` |
| `LoadBalancerPort` | ALB listener port | `443` |
| `HealthCheckPath` | Target group health check path | `/` |
| `ServiceName` | Name used for cluster, service, roles, ALB | `MyService` |
| `Image` | Container image | `public.ecr.aws/nginx/nginx:1.27.1-alpine3.20-perl` |
| `ContainerPort` | Container port | `80` |
| `Cpu` / `Memory` | vCPU / GB for the task | `0.25` / `0.5` |
| `MinContainers` / `MaxContainers` | Auto scaling range | `2` / `6` |
| `AutoScalingTargetValue` | Target average CPU (%) | `75` |
| `pTagValueApps` / `pTagValueCostCenter` | Tag values (declared but not applied to any resource) | `ecs-cluster` / `<COST_CENTER>` |

## Deploy

```bash
aws cloudformation deploy \
  --stack-name <STACK_NAME> \
  --template-file fargate-demo.yml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    VpcIdParameter=<VPC_ID> \
    ContainerSubnets=<PRIVATE_SUBNET_1>,<PRIVATE_SUBNET_2> \
    LoadbalancerSubnets=<PUBLIC_SUBNET_1>,<PUBLIC_SUBNET_2> \
    LoadBalancerCertificateArn=<CERTIFICATE_ARN> \
    ServiceName=<SERVICE_NAME>
```

## Security notes

- Overall a reasonable baseline: HTTPS only, tasks in private subnets, container SG restricted to the ALB.
- `AutoScalingRole` trusts `ecs-tasks.amazonaws.com` instead of `application-autoscaling.amazonaws.com`; the scalable target should rather rely on the Application Auto Scaling service-linked role.
- The ALB has no HTTP listener/redirect, no access logs and no deletion protection; no WAF is attached.
- The log group has no retention set (logs kept forever) and no KMS key.
- The default SSL policies offered are dated; prefer a current TLS 1.3 policy.

## Cleanup

```bash
aws cloudformation delete-stack --stack-name <STACK_NAME>
```

The log group is deleted with the stack (`DeletionPolicy: Delete`). The legacy readme file in this repo is named `Readme-md` (missing dot).

## License

MIT — see [LICENSE](LICENSE).
