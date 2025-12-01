# 🚀 **Complete Linux DevOps Learning Path - Expert Level**
*Production-Ready Implementation Guide with Best Practices*

---

## 📋 **Prerequisites**
- Ubuntu 20.04/22.04 or CentOS 7/8/RHEL
- Sudo privileges
- Minimum 2GB RAM, 20GB Disk
- Network connectivity

---
 ![alt text](<evidence/Screenshot 2025-12-01 155954.png>)
# 🔵 **LEVEL 1 – FOUNDATIONAL SKILLS**
*Mastering Core Linux Administration*

## ✅ **1. SET UP USERS & GROUPS FOR DEV TEAM**

### **Step 1: Create Secure User Management Script**
```bash
#!/bin/bash
# /usr/local/bin/setup-users.sh
# Expert-level user management with security best practices

set -e  # Exit on error

echo "🔵 LEVEL 1 - STEP 1: User & Group Setup"
echo "========================================"

# 1.1 Create DevOps team with GID 2000 (non-system range)
echo "Creating devteam group (GID: 2000)..."
sudo groupadd -g 2000 devteam
echo "✅ Group 'devteam' created with GID 2000"

# 1.2 Create users with specific UIDs and home directories
declare -A USERS=(
    ["alice"]="2001"
    ["bob"]="2002" 
    ["charlie"]="2003"
)

for USER in "${!USERS[@]}"; do
    UID=${USERS[$USER]}
    
    echo -e "\n📝 Creating user: $USER (UID: $UID)"
    
    # Check if user exists
    if id "$USER" &>/dev/null; then
        echo "⚠️  User $USER already exists, skipping..."
        continue
    fi
    
    # Create user with:
    # -m: Create home directory
    # -s /bin/bash: Set bash as default shell  
    # -u: Specific UID for consistency
    # -g devteam: Primary group
    # -G sudo: Add to sudo group for admin access
    # -c "Full Name": Comment/Full name
    sudo useradd -m \
        -s /bin/bash \
        -u "$UID" \
        -g devteam \
        -G sudo \
        -c "DevOps Engineer - $USER" \
        "$USER"
    
    # Set secure password (in production, use hashed passwords)
    echo "$USER:SecurePass$(date +%s | sha256sum | base64 | head -c 16)" | sudo chpasswd
    
    # Force password change on first login
    sudo chage -d 0 "$USER"
    
    # Set up SSH directory with secure permissions
    sudo mkdir -p /home/$USER/.ssh
    sudo chmod 700 /home/$USER/.ssh
    sudo chown -R $USER:devteam /home/$USER/.ssh
    
    echo "✅ User $USER created successfully"
done

# 1.3 Create service account for applications
echo -e "\n🔧 Creating service account: appuser"
sudo useradd -r -s /sbin/nologin -M appuser
sudo usermod -aG devteam appuser
echo "✅ Service account 'appuser' created"

# 1.4 Configure password policies
echo -e "\n🔐 Setting password policies..."
sudo apt install -y libpam-pwquality 2>/dev/null || yum install -y libpwquality 2>/dev/null

# Configure password complexity
sudo bash -c 'cat > /etc/security/pwquality.conf << EOF
minlen = 12
minclass = 3
maxrepeat = 2
maxsequence = 3
EOF'

# 1.5 Verify setup
echo -e "\n📊 VERIFICATION REPORT"
echo "====================="
echo "Group Information:"
getent group devteam
echo -e "\nUser Information:"
for USER in "${!USERS[@]}"; do
    echo "User: $USER"
    echo "  UID: $(id -u $USER)"
    echo "  Groups: $(id -Gn $USER)"
    echo "  Home: $(getent passwd $USER | cut -d: -f6)"
    echo "  Shell: $(getent passwd $USER | cut -d: -f7)"
done

echo -e "\n🔵 USER SETUP COMPLETE"
echo "Users can now log in with their credentials."
echo "First login will require password change."
```

### **Step 2: Execute User Setup**
```bash
# Make script executable
chmod +x /usr/local/bin/setup-users.sh

# Execute with logging
sudo /usr/local/bin/setup-users.sh 2>&1 | tee /var/log/user-setup-$(date +%Y%m%d).log
```

---

## ✅ **2. MANAGE PERMISSIONS FOR PROJECT DIRECTORIES**

### **Step 1: Create Advanced Permission Structure**
```bash
#!/bin/bash
# /usr/local/bin/setup-permissions.sh
# Expert-level permission management with ACL and inheritance

set -e

echo "🔵 LEVEL 1 - STEP 2: Permission Management"
echo "=========================================="

# 2.1 Install ACL utilities
echo "Installing ACL utilities..."
sudo apt install -y acl 2>/dev/null || sudo yum install -y acl 2>/dev/null

# 2.2 Create project directory structure
echo -e "\n📁 Creating project structure..."
PROJECT_BASE="/opt/projects"
sudo mkdir -p $PROJECT_BASE/{app1,app2,shared,backups,temp}

# Subdirectories with specific purposes
sudo mkdir -p $PROJECT_BASE/app1/{src,config,logs,data,bin}
sudo mkdir -p $PROJECT_BASE/app2/{src,config,logs,data,bin}
sudo mkdir -p $PROJECT_BASE/shared/{docs,assets,templates}

# 2.3 Set ownership hierarchy
echo "Setting ownership..."
sudo chown -R root:devteam $PROJECT_BASE

# 2.4 Apply base permissions (octal)
echo "Applying base permissions..."

# Project root: rwx for owner, r-x for group, --- for others
sudo chmod 750 $PROJECT_BASE

# Application directories: rwx for group
sudo chmod 770 $PROJECT_BASE/app1
sudo chmod 770 $PROJECT_BASE/app2

# Configuration files: rwx for owner, r-x for group
sudo chmod 750 $PROJECT_BASE/app1/config
sudo chmod 750 $PROJECT_BASE/app2/config

# Data directories: rwx for group with sticky bit
sudo chmod 2770 $PROJECT_BASE/app1/data  # SGID + rwx for group
sudo chmod 2770 $PROJECT_BASE/app2/data

# Log directories: rwx for group
sudo chmod 770 $PROJECT_BASE/app1/logs
sudo chmod 770 $PROJECT_BASE/app2/logs

# Shared directories: rwx for group
sudo chmod 775 $PROJECT_BASE/shared

# Temporary: rwx for all with sticky bit
sudo chmod 1777 $PROJECT_BASE/temp

# 2.5 Set ACL for granular control
echo "Setting ACLs for granular control..."

# Give alice extra permissions on app1
sudo setfacl -m u:alice:rwx $PROJECT_BASE/app1

# Give bob read-only access to app2 config
sudo setfacl -m u:bob:r-x $PROJECT_BASE/app2/config

# Default ACL for inheritance in shared
sudo setfacl -d -m g:devteam:rwx $PROJECT_BASE/shared

# 2.6 Set SGID bit for group inheritance
echo "Setting SGID for automatic group inheritance..."
sudo find $PROJECT_BASE -type d -exec chmod g+s {} \;

# 2.7 Create permission verification script
cat > /usr/local/bin/check-permissions.sh << 'EOF'
#!/bin/bash
echo "🔍 PERMISSION AUDIT REPORT"
echo "=========================="
echo "Generated: $(date)"
echo ""

for dir in /opt/projects /opt/projects/app1 /opt/projects/app2; do
    if [ -d "$dir" ]; then
        echo "Directory: $dir"
        echo "Permissions: $(stat -c '%A %U %G' $dir)"
        echo "ACL:"
        getfacl $dir | grep -v "^#" | sed 's/^/  /'
        echo ""
    fi
done

echo "User Capabilities:"
echo "alice on app1: $(getfacl /opt/projects/app1 | grep alice)"
echo "bob on app2/config: $(getfacl /opt/projects/app2/config | grep bob)"
EOF

sudo chmod +x /usr/local/bin/check-permissions.sh

# 2.8 Verification
echo -e "\n📊 VERIFICATION"
echo "=============="
/usr/local/bin/check-permissions.sh

echo -e "\n🔵 PERMISSION SETUP COMPLETE"
echo "Project structure created at: $PROJECT_BASE"
echo "Use 'getfacl' to view advanced permissions"
```

### **Step 2: Execute Permission Setup**
```bash
chmod +x /usr/local/bin/setup-permissions.sh
sudo /usr/local/bin/setup-permissions.sh
```

---

## ✅ **3. INSTALL REQUIRED PACKAGES (GIT, NGINX, JAVA)**

### **Step 1: Create Comprehensive Package Installation Script**
```bash
#!/bin/bash
# /usr/local/bin/install-packages.sh
# Expert package management with version control and verification

set -e

echo "🔵 LEVEL 1 - STEP 3: Package Installation"
echo "========================================="

LOG_FILE="/var/log/package-install-$(date +%Y%m%d).log"
exec > >(tee -a "$LOG_FILE") 2>&1

# 3.1 System detection and preparation
echo "🖥️  Detecting system..."
if [ -f /etc/os-release ]; then
    . /etc/os-release
    OS=$ID
    VER=$VERSION_ID
else
    OS=$(uname -s)
    VER=$(uname -r)
fi

echo "Operating System: $OS $VER"

# 3.2 Update package repositories
echo -e "\n🔄 Updating package repositories..."
case $OS in
    ubuntu|debian)
        sudo apt update
        sudo apt upgrade -y
        ;;
    centos|rhel|fedora)
        sudo yum update -y
        sudo yum upgrade -y
        ;;
    *)
        echo "⚠️  Unsupported OS. Manual installation required."
        exit 1
        ;;
esac

# 3.3 Install base utilities
echo -e "\n📦 Installing base utilities..."
BASE_PACKAGES="curl wget tar gzip unzip build-essential software-properties-common"

case $OS in
    ubuntu|debian)
        sudo apt install -y $BASE_PACKAGES
        ;;
    centos|rhel|fedora)
        sudo yum install -y $BASE_PACKAGES epel-release
        ;;
esac

# 3.4 Install Git with specific version
echo -e "\n🐙 Installing Git..."
case $OS in
    ubuntu|debian)
        sudo add-apt-repository ppa:git-core/ppa -y
        sudo apt update
        sudo apt install -y git git-lfs
        ;;
    centos|rhel|fedora)
        sudo yum install -y https://packages.endpointdev.com/rhel/7/main/x86_64/endpoint-repo.x86_64.rpm
        sudo yum install -y git git-lfs
        ;;
esac

# 3.5 Install Nginx with optimized configuration
echo -e "\n🌐 Installing Nginx..."
case $OS in
    ubuntu|debian)
        sudo apt install -y nginx nginx-extras
        ;;
    centos|rhel|fedora)
        sudo yum install -y nginx
        ;;
esac

# Create optimized nginx config
sudo tee /etc/nginx/nginx.conf.optimized > /dev/null << 'EOF'
user www-data;
worker_processes auto;
worker_rlimit_nofile 65535;

error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 4096;
    multi_accept on;
    use epoll;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main buffer=32k flush=5s;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    server_tokens off;

    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css application/json application/javascript text/xml;

    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
EOF

# Backup original config and apply optimized
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.backup
sudo cp /etc/nginx/nginx.conf.optimized /etc/nginx/nginx.conf

# 3.6 Install Java (OpenJDK 11 LTS)
echo -e "\n☕ Installing Java (OpenJDK 11)..."
case $OS in
    ubuntu|debian)
        sudo apt install -y openjdk-11-jdk openjdk-11-jre
        ;;
    centos|rhel|fedora)
        sudo yum install -y java-11-openjdk-devel
        ;;
esac

# 3.7 Install additional DevOps tools
echo -e "\n🛠️  Installing additional DevOps tools..."
DEVOPS_TOOLS="python3-pip nodejs npm docker.io docker-compose redis-server"

case $OS in
    ubuntu|debian)
        sudo apt install -y $DEVOPS_TOOLS
        ;;
    centos|rhel|fedora)
        sudo yum install -y $DEVOPS_TOOLS
        ;;
esac

# 3.8 Configure services
echo -e "\n⚙️  Configuring services..."

# Nginx
sudo systemctl enable nginx
sudo systemctl start nginx
sudo systemctl status nginx --no-pager

# Docker
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER

# 3.9 Verification and testing
echo -e "\n🧪 VERIFICATION TESTS"
echo "===================="

echo "1. Git Version:"
git --version

echo -e "\n2. Nginx Status:"
sudo nginx -t
sudo systemctl is-active nginx

echo -e "\n3. Java Version:"
java -version
javac -version

echo -e "\n4. Docker Status:"
docker --version
docker-compose --version
sudo systemctl is-active docker

echo -e "\n5. Python/PIP:"
python3 --version
pip3 --version

# 3.10 Create health check endpoint for nginx
echo -e "\n🔧 Creating health check endpoint..."
sudo tee /var/www/html/health.html > /dev/null << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>System Health</title>
    <style>
        body { font-family: monospace; margin: 40px; }
        .ok { color: green; }
        .error { color: red; }
    </style>
</head>
<body>
    <h1>🟢 System Status: ONLINE</h1>
    <p>Timestamp: $(date)</p>
    <p>Hostname: $(hostname)</p>
    <p>All services operational</p>
</body>
</html>
EOF

echo -e "\n🔵 PACKAGE INSTALLATION COMPLETE"
echo "All packages installed and verified"
echo "Log file: $LOG_FILE"
```

### **Step 2: Execute Package Installation**
```bash
chmod +x /usr/local/bin/install-packages.sh
sudo /usr/local/bin/install-packages.sh
```

---

## ✅ **4. CHECK SYSTEM INFO (MEMORY, CPU, DISKS)**

### **Step 1: Create Comprehensive System Monitoring Dashboard**
```bash
#!/bin/bash
# /usr/local/bin/system-dashboard.sh
# Expert system diagnostics with thresholds and alerts

set -e

echo "🔵 LEVEL 1 - STEP 4: System Information"
echo "======================================="

# Color codes for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

# Thresholds
CPU_THRESHOLD=80
MEM_THRESHOLD=85
DISK_THRESHOLD=90
LOAD_THRESHOLD=$(nproc)

# 4.1 System Identification
echo -e "${BLUE}🖥️  SYSTEM IDENTIFICATION${NC}"
echo "================================="
echo "Hostname: $(hostname -f)"
echo "Operating System: $(cat /etc/os-release | grep PRETTY_NAME | cut -d'"' -f2)"
echo "Kernel: $(uname -r)"
echo "Architecture: $(uname -m)"
echo "Uptime: $(uptime -p)"
echo "Boot Time: $(who -b | awk '{print $3, $4}')"
echo ""

# 4.2 CPU Information
echo -e "${BLUE}⚡ CPU INFORMATION${NC}"
echo "============================"
CPU_COUNT=$(nproc)
echo "CPU Cores: $CPU_COUNT"
echo "CPU Model: $(lscpu | grep "Model name" | cut -d':' -f2 | xargs)"

# CPU Usage with color coding
CPU_USAGE=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1)
if (( $(echo "$CPU_USAGE > $CPU_THRESHOLD" | bc -l) )); then
    COLOR=$RED
    STATUS="⚠️ HIGH"
elif (( $(echo "$CPU_USAGE > 50" | bc -l) )); then
    COLOR=$YELLOW
    STATUS="⚠️ MODERATE"
else
    COLOR=$GREEN
    STATUS="✅ OK"
fi
echo -e "CPU Usage: ${COLOR}${CPU_USAGE}%${NC} - $STATUS"

# Load Average
LOAD1=$(uptime | awk -F'load average:' '{print $2}' | cut -d',' -f1 | xargs)
LOAD5=$(uptime | awk -F'load average:' '{print $2}' | cut -d',' -f2 | xargs)
LOAD15=$(uptime | awk -F'load average:' '{print $2}' | cut -d',' -f3 | xargs)

echo "Load Average:"
echo "  1 min:  $LOAD1"
echo "  5 min:  $LOAD5"
echo "  15 min: $LOAD15"

if (( $(echo "$LOAD1 > $LOAD_THRESHOLD" | bc -l) )); then
    echo -e "${RED}⚠️  WARNING: High load average detected${NC}"
fi
echo ""

# 4.3 Memory Information
echo -e "${BLUE}💾 MEMORY INFORMATION${NC}"
echo "=============================="
MEM_TOTAL=$(free -h | grep Mem | awk '{print $2}')
MEM_USED=$(free -h | grep Mem | awk '{print $3}')
MEM_FREE=$(free -h | grep Mem | awk '{print $4}')
MEM_AVAIL=$(free -h | grep Mem | awk '{print $7}')
MEM_PERCENT=$(free | grep Mem | awk '{printf "%.1f", $3/$2 * 100}')

echo "Total: $MEM_TOTAL"
echo "Used:  $MEM_USED"
echo "Free:  $MEM_FREE"
echo "Available: $MEM_AVAIL"

if (( $(echo "$MEM_PERCENT > $MEM_THRESHOLD" | bc -l) )); then
    echo -e "${RED}Memory Usage: ${MEM_PERCENT}% ⚠️ HIGH${NC}"
elif (( $(echo "$MEM_PERCENT > 70" | bc -l) )); then
    echo -e "${YELLOW}Memory Usage: ${MEM_PERCENT}% ⚠️ MODERATE${NC}"
else
    echo -e "${GREEN}Memory Usage: ${MEM_PERCENT}% ✅ OK${NC}"
fi

# Swap Information
SWAP_TOTAL=$(free -h | grep Swap | awk '{print $2}')
SWAP_USED=$(free -h | grep Swap | awk '{print $3}')
echo -e "\nSwap Total: $SWAP_TOTAL"
echo "Swap Used:  $SWAP_USED"
echo ""

# 4.4 Disk Information
echo -e "${BLUE}💿 DISK INFORMATION${NC}"
echo "============================"
echo "Filesystem Overview:"
df -hT --type=ext4 --type=xfs --type=zfs --type=btrfs | grep -v tmpfs

echo -e "\n🔍 Detailed Partition Analysis:"
while IFS= read -r line; do
    if [[ $line == /dev/* ]]; then
        FS=$(echo $line | awk '{print $1}')
        SIZE=$(echo $line | awk '{print $2}')
        USED=$(echo $line | awk '{print $3}')
        AVAIL=$(echo $line | awk '{print $4}')
        USE_PERCENT=$(echo $line | awk '{print $5}' | sed 's/%//')
        MOUNT=$(echo $line | awk '{print $7}')
        
        if [ $USE_PERCENT -gt $DISK_THRESHOLD ]; then
            COLOR=$RED
            STATUS="⚠️ CRITICAL"
        elif [ $USE_PERCENT -gt 80 ]; then
            COLOR=$YELLOW
            STATUS="⚠️ WARNING"
        else
            COLOR=$GREEN
            STATUS="✅ OK"
        fi
        
        echo -e "$MOUNT ($FS): ${COLOR}${USE_PERCENT}% used${NC} - $STATUS"
        echo "  Size: $SIZE, Used: $USED, Available: $AVAIL"
    fi
done < <(df -hT)

# 4.5 Inode Information
echo -e "\n📁 Inode Usage:"
df -i | grep -v tmpfs | awk 'NR==1 || $5 > 70 {print}'

# 4.6 Network Information
echo -e "${BLUE}🌐 NETWORK INFORMATION${NC}"
echo "==============================="
echo "IP Addresses:"
ip -br addr show | awk '{print "  " $1 ": " $3}'

echo -e "\nNetwork Interfaces:"
for iface in $(ip link show | grep -E "^[0-9]+:" | awk -F: '{print $2}' | tr -d ' '); do
    if [ "$iface" != "lo" ]; then
        RX=$(ip -s link show $iface 2>/dev/null | grep -A1 "RX:" | tail -1 | awk '{print $1}')
        TX=$(ip