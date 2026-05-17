[9-1] EC2 변경 (t3.micro → t2.micro) — Phase 0 사전 정보 확인

작업일: 2026-04-22
방식: AMI 기반 Blue-Green 전환


EC2 인스턴스 (Blue — 현재 운영 중)
항목값instance-idi-08031365811d6cbb7instance-typet3.microAZap-northeast-2csubnet-idsubnet-0572ffcb598b1ff22security-group-idsg-0deaffe563d3a27b1key-pairjiong-key
EIP
항목값public-ip54.116.101.135allocation-ideipalloc-005ff5e7f9bfdfae6association-ideipassoc-044de1a9feae05fa4연결 인스턴스i-08031365811d6cbb7
IAM
항목값userjiong-iampolicyAdministratorAccess (Full)
Terraform State (module.aws)
EC2 관련

module.aws.aws_instance.nat
module.aws.aws_eip.nat (instance 직접 참조, association 리소스 없음)
module.aws.aws_security_group.nat
module.aws.aws_key_pair.jiong
module.aws.data.aws_ssm_parameter.ubuntu2404_ami

ECR

module.aws.aws_ecr_repository.main
module.aws.aws_ecr_repository.portfolio_web
module.aws.aws_ecr_repository.portfolio_web_cache

Lambda / SNS / DynamoDB (알림)

module.aws.aws_lambda_function.alert_collector
module.aws.aws_lambda_function.alert_batch_sender
module.aws.aws_sns_topic.k8s_alerts
module.aws.aws_sns_topic_subscription.alert_email
module.aws.aws_sns_topic_subscription.discord_lambda
module.aws.aws_dynamodb_table.k8s_alerts
기타 IAM role, permission, EventBridge 관련

Terraform 코드 상태 (modules/aws/main.tf)

instance_type을 t3.micro로 되돌려 실제 인프라와 일치시킴
lifecycle { ignore_changes = [ami] } 설정됨
EIP: aws_eip.nat에서 instance = aws_instance.nat.id로 직접 참조
terraform plan -target=module.aws → No changes 확인 완료

WireGuard (Bastion VM)
항목값Endpoint54.116.101.135:51820 (EIP 직접 참조)비고EIP를 Green으로 옮기면 WireGuard 자동 복구 — 추가 작업 불필요

Phase 0 완료 체크리스트

 현재 EC2 인스턴스 정보 기록
 EIP 정보 확인
 IAM 권한 확인
 Terraform modules/aws 코드 확인 + instance_type 불일치 수정
 terraform state list 기록
 Bastion VM WireGuard 설정 확인 (Endpoint: EIP 직접)
