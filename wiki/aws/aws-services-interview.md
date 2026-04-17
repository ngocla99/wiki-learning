# AWS Services Interview Q&A

> Sources: AWS Documentation; practical LMP project experience
> Updated: 2026-04-17

## Overview

Tổng hợp các câu hỏi phỏng vấn phổ biến về AWS cloud computing, tập trung vào các services thực tế đang dùng trong project: EC2, RDS, IAM, S3, CloudWatch, VPC, CodeBuild. Mỗi câu trả lời ngắn gọn, đủ để trả lời phỏng vấn fullstack developer level.

---

## EC2 — Máy chủ ảo

### EC2 là gì? Dùng để làm gì?

EC2 (Elastic Compute Cloud) là máy chủ ảo trên AWS. Dùng để chạy application — backend Node.js, frontend React build, hoặc bất kỳ phần mềm nào cần server.

> Trong project: 4 EC2 instances chạy cho 4 môi trường — test (`t3a.small`), stage (`t3a.small`), UAT (`t3a.small`), management (`t3a.large`). Tất cả ở region `us-east-1`.

### Làm sao để kết nối (SSH) vào EC2?

AWS cung cấp 4 cách kết nối vào EC2 instance:

| Cách | Cần gì | Khi nào dùng |
|------|--------|-------------|
| **SSH Client** | Key pair (`.pem` file) + Public IP | Phổ biến nhất, dùng terminal |
| **EC2 Instance Connect** | Chỉ cần browser, không cần key | Nhanh, tiện, không cần setup |
| **SSM Session Manager** | IAM Role có SSM permission | An toàn nhất, không cần mở port 22 |
| **EC2 Serial Console** | Password trên OS | Debug khi instance không kết nối được |

**Cách team đang dùng — SSH Client**:

```bash
# 1. Có file key pair: lms-keypair.pem
# 2. Set permission (Linux/Mac)
chmod 400 "lms-keypair.pem"

# 3. SSH vào instance bằng Public IP
ssh -i "lms-keypair.pem" ec2-user@54.204.125.9
```

> Trong project: Instance UAT `ec2-ewoo-lms-uat-ue1` dùng key pair `lms-keypair.pem`, user `ec2-user`, IAM role `ec2_codebuild`, Security Group `sg-07f69be10b78ff54a (default)`, VPC `vpc-0f8f33887a31a0150`.

**Giải thích từng thành phần**:
- **Key Pair (`lms-keypair.pem`)**: File private key. AWS giữ public key trên instance. Ai có file này mới SSH được → **phải bảo mật, không commit lên git**.
- **`ec2-user`**: Username mặc định của Amazon Linux AMI. Ubuntu dùng `ubuntu`, Debian dùng `admin`.
- **Public IP**: IP công khai để kết nối từ internet. Thay đổi khi stop/start instance (trừ khi dùng Elastic IP).
- **IAM Role `ec2_codebuild`**: Role gắn vào instance, cho phép instance access các AWS services (S3, CodeBuild) mà không cần lưu access keys trên server.

**SSM Session Manager** (cách an toàn hơn):
- Không cần mở port 22 trong Security Group.
- Không cần key pair.
- Kết nối qua AWS console hoặc CLI: `aws ssm start-session --target i-0c980c5f42f30ad87`.
- Mọi session được log lại → audit trail.

### Security Group là gì?

Firewall ảo ở mức instance. Chỉ cho phép traffic theo rules bạn define (port, protocol, source IP). **Stateful** — cho phép inbound thì response tự động được phép outbound.

---

## RDS — Database được quản lý

### RDS là gì? Dùng để làm gì?

RDS (Relational Database Service) là database được AWS quản lý — AWS lo việc backup, patching, và high availability. Hỗ trợ MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, và Aurora.

> Trong project: 3 RDS instances — `db-ewoo-lms-test-ue1`, `db-ewoo-lms-stage-ue1`, `db-ewoo-lms-uat-ue1`. Mỗi môi trường có database riêng.

### RDS khác gì tự cài database trên EC2?

| RDS (Managed) | Self-managed trên EC2 |
|---|---|
| AWS lo backup, patching, failover | Tự làm tất cả |
| Multi-AZ 1 click | Tự setup replication |
| Ít control hơn (không SSH vào được) | Full control |
| Tốn tiền hơn một chút | Rẻ hơn nhưng tốn effort |

### Multi-AZ là gì?

AWS tự tạo 1 bản sao database ở Availability Zone khác. Nếu AZ chính chết → tự động failover sang AZ backup trong 60-120 giây. **Mục đích**: High availability, không phải performance.

### Read Replica là gì?

Bản sao chỉ đọc (read-only) của database chính. **Mục đích**: Giảm tải read queries (reports, analytics) cho database chính.

---

## IAM — Quản lý quyền truy cập

### IAM là gì? Dùng để làm gì?

IAM (Identity and Access Management) quản lý **ai** được phép làm **gì** trên AWS. Gồm Users, Groups, Roles, và Policies.

> Trong project: 11 IAM users — `fpt_admin` (admin), `fpt_dev` (developer), `fpt_devops` (CI/CD, 3 access keys), `BespinSupport` (external support), v.v.

### User vs Role khác gì nhau?

- **User**: Tài khoản cố định, có username/password hoặc access keys (long-term credentials). Cho **người** dùng.
- **Role**: Credentials tạm thời, không có password. Cho **services** (EC2, Lambda) hoặc cross-account access.

**Best practice**: EC2 instance nên dùng **IAM Role** thay vì lưu access keys trên server.

### Policy là gì?

JSON document quy định permissions. Gồm: Effect (Allow/Deny), Action (s3:GetObject), Resource (bucket cụ thể).

**Nguyên tắc đánh giá**: Explicit Deny > Explicit Allow > Default Deny.

### Câu hỏi bảo mật thường gặp?

- **Không bao giờ** dùng root account cho daily work.
- Bật **MFA** cho root và tất cả users.
- Dùng **least privilege** — chỉ cấp quyền tối thiểu cần thiết.
- **Rotate access keys** định kỳ. Xóa keys không sử dụng.

> Lưu ý: Trong project có users không active lâu (certbot-dns-route53: 1296 ngày, payton: 1186 ngày, test-acc: 1190 ngày). Đây là security risk — nên review và disable/xóa.

---

## S3 — Lưu trữ đối tượng

### S3 là gì? Dùng để làm gì?

S3 (Simple Storage Service) lưu trữ files (objects) với độ bền 99.999999999% (11 nines). Dùng để lưu: installers, static assets, backups, logs, media files.

> Trong project: Bucket `s3-release-lms-stage-ue1` lưu các **phiên bản cài đặt (installers)** của các ứng dụng desktop mà công ty phát triển: EzDent, Ez3D-i, EzGate, EzOrtho, EzServer, DAVIS Toolkit, CleverOne, v.v. Khi có release mới → upload installer lên S3 → support/khách hàng tải về cài đặt. S3 **không** dùng cho web app deployment trong project này.

### S3 dùng cho React app như thế nào?

**Host static SPA**: Build React app → upload lên S3 → đặt CloudFront phía trước → có website nhanh, rẻ, scale vô hạn.

> Lưu ý: Trong project LMP, web app được deploy qua Azure DevOps self-hosted agent trực tiếp lên EC2, không đi qua S3.

### Storage Classes phổ biến?

| Class | Dùng khi |
|---|---|
| Standard | Data truy cập thường xuyên |
| Standard-IA | Data ít truy cập, nhưng cần lấy nhanh |
| Glacier | Archive, lưu lâu dài, chấp nhận lấy chậm |

### Pre-signed URL là gì?

URL tạm thời cho phép ai đó download/upload file private trên S3 mà không cần AWS credentials. Có thời hạn (vd: 1 giờ).

---

## CloudWatch — Monitoring & Logging

### CloudWatch là gì? Dùng để làm gì?

CloudWatch thu thập **metrics** (CPU, memory, request count) và **logs** (application logs, build logs) từ AWS resources. Dùng để monitor, debug, và set up alerts.

> Trong project: Log group `/aws/codebuild/license-app` lưu build logs từ CodeBuild. Retention đang set "Never expire" — nên cân nhắc set retention 30-90 ngày để tiết kiệm.

### CloudWatch Logs hoạt động thế nào?

- **Log Group**: Nhóm logs theo app/service (vd: `/aws/codebuild/license-app`).
- **Log Stream**: Mỗi instance/build run tạo 1 stream.
- Có thể search, filter logs bằng **Logs Insights** (query language).

### CloudWatch Alarms dùng để làm gì?

Cảnh báo khi metric vượt ngưỡng. Ví dụ:
- EC2 CPU > 80% → gửi email qua SNS.
- Số lỗi 5xx tăng đột biến → notify team.
- Kết hợp với Auto Scaling: alarm trigger → thêm/bớt instances.

---

## VPC — Mạng ảo

### VPC là gì? Dùng để làm gì?

VPC (Virtual Private Cloud) là mạng ảo riêng trên AWS, nơi bạn đặt tất cả resources (EC2, RDS, etc.). Giống như bạn có 1 data center riêng trên cloud.

### Public vs Private Subnet?

- **Public Subnet**: Có kết nối internet trực tiếp. Đặt: Load Balancer, bastion host.
- **Private Subnet**: Không có internet trực tiếp. Đặt: EC2 app servers, RDS databases. Muốn ra internet (cập nhật, gọi API) → đi qua **NAT Gateway**.

**Best practice**: Backend servers và databases **luôn** đặt trong private subnet.

### Security Group vs NACL?

| Security Group | NACL |
|---|---|
| Ở mức instance | Ở mức subnet |
| Stateful (tự cho response) | Stateless (phải cho cả 2 chiều) |
| Chỉ có Allow rules | Có cả Allow và Deny |

---

## CodeBuild — Build trên cloud

### CodeBuild là gì? Dùng để làm gì?

CodeBuild compile code, chạy tests, và tạo build artifacts trên cloud. Không cần quản lý build server — AWS lo.

> Trong project: CodeBuild build `license-app`, logs ghi vào CloudWatch log group `/aws/codebuild/license-app`.

### CodeBuild khác gì Azure DevOps Pipelines build?

| CodeBuild | Azure DevOps Build |
|---|---|
| Chạy trên AWS, gần resources | Chạy trên Azure/Microsoft agents |
| Tính tiền theo phút build | Có free tier, tính theo parallel jobs |
| Tích hợp sâu AWS (IAM, S3, ECR) | Tích hợp sâu Azure, nhưng có AWS extensions |

**Lý do dùng cả hai**: Azure DevOps quản lý release workflow (approval, stages). CodeBuild chạy build trên AWS để access AWS resources nhanh hơn và tiết kiệm data transfer.

---

## Câu hỏi tổng hợp thường gặp

### Giải thích flow deploy trong project của bạn?

```
1. Developer merge code vào branch
2. Azure DevOps Build Pipeline tự động trigger
   → Build ra 2 artifacts: _[LMP].App (backend) + _[LMP].Web (frontend React)
   → Mỗi build có version (vd: 260416.01)
3. Khi muốn deploy: vào Azure DevOps Releases → Create Release
   → Chọn build version cho App và Web
   → Chọn stage muốn deploy (Test/Staging/UAT/Production)
4. Release trigger job → Self-hosted Agent trên EC2 mgmt nhận job
   (Agent Pool: "Management Environment Agent Pool")
   (Agent chạy trên: ec2-ewoo-lms-mgmt-ue1, t3a.large)
5. Agent thực thi deployment scripts → deploy lên EC2 target tương ứng
6. CloudWatch thu thập logs để monitor
```

### Self-hosted Agent là gì? Tại sao dùng?

EC2 management server (`ec2-ewoo-lms-mgmt-ue1`, `t3a.large`) cài đặt **Azure DevOps Agent** — phần mềm nhận và thực thi deployment jobs từ Azure DevOps.

| Microsoft-hosted agent | Self-hosted agent (đang dùng) |
|---|---|
| Chạy trên Azure cloud | Chạy trên EC2 trong cùng VPC |
| Không access được private network | **Truy cập trực tiếp** EC2/RDS trong VPC |
| Mỗi job tạo VM mới | Persistent, nhanh hơn |
| Tính tiền parallel jobs | Miễn phí (tự quản lý) |

**Lý do dùng**: Agent nằm cùng VPC → SSH trực tiếp sang các EC2 target (test/stage/uat) mà không cần expose ra internet. Đây cũng là lý do con mgmt dùng `t3a.large` (nhiều RAM hơn) — vì nó chạy agent + xử lý deployment.

### Tại sao dùng nhiều environments (Test/Stage/UAT/Production)?

- **Test**: Developer test, tự do deploy, có thể break.
- **Staging**: Giống production nhất có thể, QA test.
- **UAT**: User/client acceptance testing.
- **Production**: End users sử dụng.

Mỗi environment có EC2 riêng, RDS riêng → isolate hoàn toàn, tránh ảnh hưởng lẫn nhau.

### So sánh AWS vs Azure? Tại sao dùng cả hai?

| AWS | Azure |
|---|---|
| Infrastructure (compute, storage, DB) | CI/CD pipeline, project management |
| EC2, RDS, S3, CloudWatch | Azure DevOps (Boards, Repos, Pipelines, Releases) |
| Mạnh về cloud services | Mạnh về DevOps toolchain |

**Lý do kết hợp**: Azure DevOps quản lý CI/CD workflow (build pipeline, release stages, approval gates). AWS cung cấp infrastructure (EC2, RDS, VPC). Self-hosted agent trên EC2 mgmt là cầu nối — nhận lệnh từ Azure DevOps, deploy lên AWS.

### AWS Region và Availability Zone là gì?

- **Region**: Khu vực địa lý (vd: `us-east-1` = N. Virginia). Chọn gần users để giảm latency.
- **Availability Zone (AZ)**: Data center độc lập trong 1 Region (vd: `us-east-1a`, `us-east-1b`). Dùng nhiều AZ để high availability.

> Trong project: Tất cả resources đều ở `us-east-1a`.

### Nếu hỏi về cost optimization?

- **Right-sizing**: Dùng instance type phù hợp (t3a.small cho dev/test, không cần m5.large).
- **Reserved Instances**: Commit 1-3 năm cho production → tiết kiệm đến 72%.
- **S3 Lifecycle**: Chuyển old artifacts sang storage class rẻ hơn.
- **CloudWatch Retention**: Set log retention thay vì "Never expire".
- **Xóa resources không dùng**: Tắt EC2 ngoài giờ làm việc cho dev/test environments.
