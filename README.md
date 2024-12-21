Dependencies Installation

# Install dependencies for building from source
sudo apt update
sudo apt install -y curl git jq lz4 build-essential

# Install Go
sudo rm -rf /usr/local/go
curl -L https://go.dev/dl/go1.22.7.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.profile
source .profile
Node Installation

Node Name

Your Node Name
Port prefix

232
# Clone project repository
cd && rm -rf chain
git clone https://github.com/UnUniFi/chain
cd chain
git checkout v4.0.2

# Build binary
make install

# Prepare cosmovisor directories
mkdir -p $HOME/.ununifi/cosmovisor/genesis/bin
ln -s $HOME/.ununifi/cosmovisor/genesis $HOME/.ununifi/cosmovisor/current -f

# Copy binary to cosmovisor directory
cp $(which ununifid) $HOME/.ununifi/cosmovisor/genesis/bin

# Set node CLI configuration
ununifid config chain-id ununifi-beta-v1
ununifid config keyring-backend file
ununifid config node tcp://localhost:23257

# Initialize the node
ununifid init "Your Node Name" --chain-id ununifi-beta-v1

# Download genesis and addrbook files
curl -L https://snapshots.nodejumper.io/ununifi/genesis.json > $HOME/.ununifi/config/genesis.json
curl -L https://snapshots.nodejumper.io/ununifi/addrbook.json > $HOME/.ununifi/config/addrbook.json

# Set seeds
sed -i -e 's|^seeds *=.*|seeds = "8542cd7e6bf9d260fef543bc49e59be5a3fa9074@seed.publicnode.com:26656,fa38d2a851de43d34d9602956cd907eb3942ae89@a.ununifi.cauchye.net:26656,404ea79bd31b1734caacced7a057d78ae5b60348@b.ununifi.cauchye.net:26656,ebc272824924ea1a27ea3183dd0b9ba713494f83@ununifi-mainnet-seed.autostake.com:26746,1357ac5cd92b215b05253b25d78cf485dd899d55@[2600:1f1c:534:8f02:7bf:6b31:3702:2265]:26656,25006d6b85daeac2234bcb94dafaa73861b43ee3@[2600:1f1c:534:8f02:a407:b1c6:e8f5:94b]:26656,caf792ed396dd7e737574a030ae8eabe19ecdf5c@[2600:1f1c:534:8f02:b0a4:dbf6:e50b:d64e]:26656,796c62bb2af411c140cf24ddc409dff76d9d61cf@[2600:1f1c:534:8f02:ca0e:14e9:8e60:989e]:26656,cea8d05b6e01188cf6481c55b7d1bc2f31de0eed@[2600:1f1c:534:8f02:ba43:1f69:e23a:df6b]:26656,20e1000e88125698264454a884812746c2eb4807@seeds.lavenderfive.com:23256"|' $HOME/.ununifi/config/config.toml

# Set minimum gas price
sed -i -e 's|^minimum-gas-prices *=.*|minimum-gas-prices = "0.0025uguu"|' $HOME/.ununifi/config/app.toml

# Set pruning
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "17"|' \
  $HOME/.ununifi/config/app.toml

# Enable prometheus
sed -i -e 's|^prometheus *=.*|prometheus = true|' $HOME/.ununifi/config/config.toml

# Change ports
sed -i -e "s%:1317%:23217%; s%:8080%:23280%; s%:9090%:23290%; s%:9091%:23291%; s%:8545%:23245%; s%:8546%:23246%; s%:6065%:23265%" $HOME/.ununifi/config/app.toml
sed -i -e "s%:26658%:23258%; s%:26657%:23257%; s%:6060%:23260%; s%:26656%:23256%; s%:26660%:23261%" $HOME/.ununifi/config/config.toml

# Download latest chain data snapshot
curl "https://snapshots.nodejumper.io/ununifi/ununifi_latest.tar.lz4" | lz4 -dc - | tar -xf - -C "$HOME/.ununifi"

# Install Cosmovisor
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.7.0

# Create a service
sudo tee /etc/systemd/system/ununifi.service > /dev/null << EOF
[Unit]
Description=UnUniFi node service
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.ununifi
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.ununifi"
Environment="DAEMON_NAME=ununifid"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=true"
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable ununifi.service

# Start the service and check the logs
sudo systemctl start ununifi.service
sudo journalctl -u ununifi.service -f --no-hostname -o cat
Secure Server Setup (Optional)

# generate ssh keys, if you don't have them already, DO IT ON YOUR LOCAL MACHINE
ssh-keygen -t rsa

# save the output, we'll use it later on instead of YOUR_PUBLIC_SSH_KEY
cat ~/.ssh/id_rsa.pub
# upgrade system packages
sudo apt update
sudo apt upgrade -y

# add new admin user
sudo adduser admin --disabled-password -q

# upload public ssh key, replace YOUR_PUBLIC_SSH_KEY with the key above
mkdir /home/admin/.ssh
echo "YOUR_PUBLIC_SSH_KEY" >> /home/admin/.ssh/authorized_keys
sudo chown admin: /home/admin/.ssh
sudo chown admin: /home/admin/.ssh/authorized_keys

echo "admin ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

# disable root login, disable password authentication, use ssh keys only
sudo sed -i 's|^PermitRootLogin .*|PermitRootLogin no|' /etc/ssh/sshd_config
sudo sed -i 's|^ChallengeResponseAuthentication .*|ChallengeResponseAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PasswordAuthentication .*|PasswordAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PermitEmptyPasswords .*|PermitEmptyPasswords no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PubkeyAuthentication .*|PubkeyAuthentication yes|' /etc/ssh/sshd_config

sudo systemctl restart sshd

# install fail2ban
sudo apt install -y fail2ban

# install and configure firewall
sudo apt install -y ufw
sudo ufw default allow outgoing
sudo ufw default deny incoming
sudo ufw allow ssh
sudo ufw allow 9100
sudo ufw allow 26656

# make sure you expose ALL necessary ports, only after that enable firewall
sudo ufw enable

# make terminal colorful
sudo su - admin
source <(curl -s https://raw.githubusercontent.com/nodejumper-org/cosmos-scripts/master/utils/enable_colorful_bash.sh)

# update servername, if needed, replace YOUR_SERVERNAME with wanted server name
sudo hostnamectl set-hostname YOUR_SERVERNAME

# now you can logout (exit) and login again using ssh admin@YOUR_SERVER_IP
