## 4. Set up AWS EC2 (one time)

1. **Launch an instance** in the EC2 console:
   - AMI: **Amazon Linux 2023**
   - Type: `t2.micro` or `t3.micro` (free tier eligible)
   - Key pair: create a new one and download the `.pem` file
   - Security group inbound rules:
     - **HTTP (80)** from `0.0.0.0/0` (so anyone can see the app)
     - **SSH (22)** from `0.0.0.0/0` (GitHub-hosted runners use changing IPs; see *Security notes*)
   - Optional: paste the contents of `deploy/ec2-setup.sh` into **Advanced details → User data** to skip step 3.

2. **Run the setup script** (skip if you used User data):
   ```bash
     vim 
     paste the script content and save
   ```
   **Make the script executable
   chmod 755 ec2-setup.sec2-setup.shh

   **Run the script
   sudo su
   ./ec2-setup.sh


   Visiting `http://<EC2_PUBLIC_IP>` now shows a *502 Bad Gateway*. That's expected: nginx is running but no app has been deployed yet.

## 5. Connect GitHub to EC2 (one time)

1. Create a GitHub repository and push this project to the `main` branch:
   ```bash
   git init -b main
   git add .
   git commit -m "Initial commit"
   git remote add origin https://github.com/<you>/<repo>.git
   git push -u origin main
   ```
2. In the repo go to **Settings → Secrets and variables → Actions**:

   | Kind | Name | Value |
   |---|---|---|
   | **Secret** | `EC2_SSH_KEY` | Entire contents of your `.pem` file, including the `BEGIN`/`END` lines |
   | **Variable** | `EC2_HOST` | Public IP or DNS of the instance, e.g. `54.12.34.56` |
   | **Variable** | `EC2_USER` | `ec2-user` |

# Create a GitHub Repository
test-sldc-deployment
git init
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin git@github.com:Ernest41k/test-sldc-deployment.git
git push -u origin main