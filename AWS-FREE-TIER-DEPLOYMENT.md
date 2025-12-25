# Deployment ke AWS Free Tier dengan OpenAI

Panduan lengkap untuk deploy n8n di AWS EC2 Free Tier menggunakan OpenAI sebagai AI provider.

## Prerequisites

1. Akun AWS dengan Free Tier aktif
2. OpenAI API Key dari https://platform.openai.com/api-keys
3. Credit card terdaftar di AWS (required meski free tier)

## Step 1: Buat EC2 Instance

1. Login ke AWS Console: https://ap-southeast-2.console.aws.amazon.com/ec2/
2. Klik **"Launch Instance"**
3. Konfigurasi:

   - **Name**: n8n-free-tier
   - **AMI**: Ubuntu Server 22.04 LTS (Free tier eligible)
   - **Instance type**: t2.micro (1 vCPU, 1GB RAM)
   - **Key pair**: Create new atau pilih existing
   - **Network settings**:
     - Allow SSH (port 22)
     - Add rule: Custom TCP, Port 5678, Source: 0.0.0.0/0 (untuk n8n)
   - **Storage**: 20-30GB gp3 (dalam free tier limit)

4. Klik **"Launch Instance"**

## Step 2: Connect ke EC2 Instance

### Via SSH (Windows PowerShell):

```powershell
ssh -i "path/to/your-key.pem" ubuntu@your-ec2-public-ip
```

### Via AWS Console:

- Klik instance → Connect → EC2 Instance Connect → Connect

## Step 3: Install Docker & Docker Compose

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker

# Add user to docker group
sudo usermod -aG docker ubuntu
newgrp docker

# Install Docker Compose
sudo apt install docker-compose -y

# Verify installation
docker --version
docker-compose --version
```

## Step 4: Deploy n8n

```bash
# Clone repository
git clone https://github.com/n8n-io/self-hosted-ai-starter-kit.git
cd self-hosted-ai-starter-kit

# Copy environment file
cp .env.free-tier.example .env

# Edit .env file
nano .env
```

### Update .env dengan:

```bash
# Generate random strings untuk encryption keys
POSTGRES_PASSWORD=$(openssl rand -base64 32)
N8N_ENCRYPTION_KEY=$(openssl rand -base64 32)
N8N_USER_MANAGEMENT_JWT_SECRET=$(openssl rand -base64 32)

# Get EC2 public IP
EC2_IP=$(curl -s http://checkip.amazonaws.com)

# Update .env
N8N_HOST=http://$EC2_IP:5678
WEBHOOK_URL=http://$EC2_IP:5678/

# Add your OpenAI API Key
OPENAI_API_KEY=sk-your-actual-api-key-here
```

### Start services:

```bash
# Start dengan config free tier
docker-compose -f docker-compose.free-tier.yml up -d

# Check logs
docker-compose -f docker-compose.free-tier.yml logs -f n8n

# Check status
docker-compose -f docker-compose.free-tier.yml ps
```

## Step 5: Access n8n

1. Buka browser: `http://your-ec2-public-ip:5678`
2. Buat akun owner pertama kali
3. Login

## Step 6: Setup OpenAI di n8n

1. Di n8n, klik **Settings** (kiri bawah) → **Credentials**
2. Klik **"Add Credential"**
3. Cari dan pilih **"OpenAI"**
4. Masukkan API Key Anda
5. Klik **"Save"**

## Step 7: Test dengan Workflow

1. Create New Workflow
2. Add node **"Manual"** trigger
3. Add node **"OpenAI Chat Model"**
   - Select credential yang tadi dibuat
   - Model: gpt-3.5-turbo atau gpt-4
4. Add node **"AI Agent"**
5. Connect nodes dan test execute

## Maintenance Commands

```bash
# Stop services
docker-compose -f docker-compose.free-tier.yml down

# Restart services
docker-compose -f docker-compose.free-tier.yml restart

# View logs
docker-compose -f docker-compose.free-tier.yml logs -f

# Update n8n to latest version
docker-compose -f docker-compose.free-tier.yml pull
docker-compose -f docker-compose.free-tier.yml up -d

# Backup data
docker run --rm -v self-hosted-ai-starter-kit_n8n_storage:/data -v $(pwd):/backup ubuntu tar czf /backup/n8n-backup-$(date +%Y%m%d).tar.gz /data
```

## Resource Monitoring

```bash
# Check memory usage
free -h

# Check disk usage
df -h

# Check Docker stats
docker stats

# If running out of memory, restart services
docker-compose -f docker-compose.free-tier.yml restart
```

## Cost Optimization Tips

1. **Stop instance saat tidak digunakan** (Free tier = 750 jam/bulan)

   ```bash
   # Di AWS Console: Instance → Instance State → Stop
   ```

2. **Setup CloudWatch Alarm** untuk monitor biaya

3. **Limit OpenAI usage**:

   - Set spending limits di OpenAI dashboard
   - Use gpt-3.5-turbo instead of gpt-4 (lebih murah)

4. **Backup regularly** sebelum stop instance

## Troubleshooting

### Port 5678 tidak bisa diakses:

```bash
# Check security group di AWS Console
# Pastikan port 5678 open untuk 0.0.0.0/0

# Check n8n running
docker ps | grep n8n

# Check logs
docker logs n8n
```

### Out of Memory:

```bash
# Add swap space
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### Docker container crashed:

```bash
# Restart all services
docker-compose -f docker-compose.free-tier.yml restart

# If still issues, recreate containers
docker-compose -f docker-compose.free-tier.yml down
docker-compose -f docker-compose.free-tier.yml up -d
```

## Security Recommendations

1. **Setup firewall** (UFW):

```bash
sudo ufw allow 22/tcp
sudo ufw allow 5678/tcp
sudo ufw enable
```

2. **Change default ports** di docker-compose (optional)

3. **Setup domain & SSL** dengan Let's Encrypt (recommended untuk production)

4. **Regular updates**:

```bash
sudo apt update && sudo apt upgrade -y
```

## Estimated Monthly Costs

- **EC2 t2.micro**: $0 (dalam free tier 750 jam/bulan untuk 12 bulan pertama)
- **EBS Storage 30GB**: $0 (dalam free tier)
- **Data Transfer**: $0 (15GB/bulan dalam free tier)
- **OpenAI API**: Varies (tergantung usage, start dari ~$5-20/bulan)

**Total**: ~$5-20/bulan (hanya OpenAI, AWS free)

Setelah 12 bulan, EC2 t2.micro ~$8-10/bulan.

## Next Steps

- Setup automatic backups
- Configure domain name (Route53 atau external)
- Add SSL certificate (Let's Encrypt)
- Setup monitoring (CloudWatch)
- Explore n8n templates: https://n8n.io/workflows
