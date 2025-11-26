# StrongSwan 6.0.2 IPsec VPN Setup Guide

This guide demonstrates setting up an IPsec VPN using StrongSwan 6.0.2 with OpenSSL 3.5.3 on Ubuntu 22.04. The implementation runs IPsec VPN in tunnel mode with configurable authentication methods: X.509 certificates or pre-shared keys.

The setup supports two operational modes:

**Classical Mode:**

- RSA-4096 certificates for authentication
- Traditional key exchange algorithms (Classical certs and RSA key exchange)

**Post-Quantum Mode:**

- Pre-shared keys: ML-KEM key exchange groups
- Certificate authentication: RSA-4096 certificates with ML-KEM key exchange

> [!NOTE]
> StrongSwan v6.0.2 currently does not support post-quantum certificates such as ML-DSA. Development is in progress at https://github.com/strongswan/strongswan/tree/ml-dsa

## Prerequisites

- Ubuntu 22.04 LTS
- Root or sudo access
- Two machines: one for server, one for client

## System Preparation (for both machines)

### 1. Install Build Dependencies

```bash
sudo apt-get update
sudo apt-get install -y \
    build-essential \
    git \
    wget \
    curl \
    autoconf \
    automake \
    libtool \
    pkg-config \
    gettext \
    bison \
    flex \
    gperf \
    libgmp-dev \
    libcurl4-openssl-dev \
    libsystemd-dev \
    libpam0g-dev \
    libldap2-dev \
    libsqlite3-dev \
    libmysqlclient-dev \
    libpq-dev \
    libxml2-dev \
    libjson-c-dev \
    libcap-dev \
    libiptc-dev \
    libnm-dev \
    libxtables-dev \
    libip4tc-dev \
    libip6tc-dev \
    libnetfilter-conntrack-dev \
    iproute2 \
    iputils-ping \
    net-tools \
    tcpdump \
    python3 \
    vim \
    checkinstall \
    zlib1g-dev \
    perl-modules-5.34 \
    perl-doc
```

### 2. Build and Install OpenSSL 3.5.3

```bash
cd /tmp
wget https://github.com/openssl/openssl/releases/download/openssl-3.5.3/openssl-3.5.3.tar.gz
tar -xzf openssl-3.5.3.tar.gz
cd openssl-3.5.3
./config --prefix=/usr/local --openssldir=/usr/local/ssl
make -j$(nproc)
sudo make install
sudo ldconfig
```

Verify installation:

```bash
LD_LIBRARY_PATH=/usr/local/lib64 openssl version
```

> [!NOTE]  
> Don't make a global configuration change (e.g. at `/etc/ld.so.conf.d/custom-openssl.conf`) as it would force your system tools (like SSH, apt, and curl) to try loading your custom OpenSSL 3.5.3, which is incompatible with the version they were built against (OpenSSL 3.0.x or less).

### 3. Build and Install StrongSwan 6.0.2

```bash
cd /tmp
wget --no-check-certificate https://download.strongswan.org/strongswan-6.0.2.tar.bz2
tar xjf strongswan-6.0.2.tar.bz2
cd strongswan-6.0.2

PKG_CONFIG_PATH="/usr/local/lib64/pkgconfig:/usr/local/lib/pkgconfig" \
CPPFLAGS="-I/usr/local/include" \
LDFLAGS="-L/usr/local/lib64 -L/usr/local/lib -Wl,-rpath,/usr/local/lib64 -Wl,-rpath,/usr/local/lib" \
./configure \
    --prefix=/usr \
    --sysconfdir=/etc \
    --localstatedir=/var \
    --with-systemdsystemunitdir=/lib/systemd/system \
    --disable-defaults \
    --enable-charon \
    --enable-systemd \
    --enable-vici \
    --enable-swanctl \
    --enable-ikev2 \
    --enable-openssl \
    --enable-ml \
    --enable-nonce \
    --enable-random \
    --enable-pem \
    --enable-x509 \
    --enable-pkcs1 \
    --enable-pkcs8 \
    --enable-pki \
    --enable-pubkey \
    --enable-socket-default \
    --enable-kernel-netlink \
    --enable-updown \
    --enable-resolve \
    --enable-silent-rules

make -j$(nproc)
sudo make install
sudo ldconfig
```

### 4. Create Required Directories

```bash
sudo mkdir -p /etc/swanctl/conf.d \
             /etc/swanctl/x509ca \
             /etc/swanctl/x509 \
             /etc/swanctl/private \
             /etc/swanctl/rsa \
             /etc/swanctl/ecdsa \
             /etc/swanctl/pkcs8 \
             /etc/swanctl/pkcs12 \
             /etc/swanctl/pubkey

sudo chmod 700 /etc/swanctl/private /etc/swanctl/rsa /etc/swanctl/ecdsa /etc/swanctl/pkcs8 /etc/swanctl/pkcs12
sudo chown -R root:root /etc/swanctl
```

### 5. Enable IP Forwarding

```bash
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -w net.ipv4.conf.all.accept_redirects=0
sudo sysctl -w net.ipv4.conf.all.send_redirects=0

# Make permanent
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## PKI Generation

Use the provided script to generate certificates.

```bash
git clone https://github.com/chmodshubham/strongswan.git
cd strongswan/
chmod +x generate-pki.sh
./generate-pki.sh <IPSEC_MODE>
```

> [!NOTE]  
> **IPSEC_MODE** can be either `classical` or `pq-support` and **AUTH_MODE** can be either `certs` for certificate-based authentication or `psk` for pre-shared key authentication.

## Server Configuration

### 1. Copy Certificates (only for certificate-based auth)

```bash
sudo cp <IPSEC_MODE>/pki/cacerts/ca-cert.pem /etc/swanctl/x509ca/
sudo cp <IPSEC_MODE>/pki/certs/server-cert.pem /etc/swanctl/x509/
sudo cp <IPSEC_MODE>/pki/private/server-key.pem /etc/swanctl/private/
sudo cp <IPSEC_MODE>/pki/certs/client-cert.pem /etc/swanctl/x509/
sudo cp <IPSEC_MODE>/pki/private/client-key.pem /etc/swanctl/private/
```

### 2. Copy Configuration Files

Update the `local_addrs` and `remote_addrs` in the respective `<IPSEC_MODE>/conf/<AUTH_MODE>/server.conf` file. Copy the server configuration from your repository:

```bash
sudo cp <IPSEC_MODE>/conf/<AUTH_MODE>/server.conf /etc/swanctl/swanctl.conf
# sudo cp <IPSEC_MODE>/conf/psk/secrets.conf /etc/ipsec.secrets
```

> [!TIP]
> `swanctl` completely ignores `/etc/ipsec.secrets`. It only reads secrets defined inside `/etc/swanctl/swanctl.conf`.

### 3. Configure Server Network

```bash
# Create tunnel interface
sudo ip link add name tunnel0 type dummy
sudo ip addr add 10.1.0.1/32 dev tunnel0
sudo ip link set tunnel0 up
```

### 4. Start StrongSwan

```bash
# Start the daemon
sudo systemctl enable strongswan
sudo systemctl start strongswan
```

Verify Swanctl Status

```bash
sudo swanctl --version
```

### 5. Load Configuration

```bash
sudo swanctl --load-all
```

## Client Configuration

### 1. Copy Certificates (only for certificate-based auth)

```bash
sudo cp <IPSEC_MODE>/pki/cacerts/ca-cert.pem /etc/swanctl/x509ca/
sudo cp <IPSEC_MODE>/pki/certs/server-cert.pem /etc/swanctl/x509/
sudo cp <IPSEC_MODE>/pki/private/server-key.pem /etc/swanctl/private/
sudo cp <IPSEC_MODE>/pki/certs/client-cert.pem /etc/swanctl/x509/
sudo cp <IPSEC_MODE>/pki/private/client-key.pem /etc/swanctl/private/
```

### 2. Copy Configuration Files


Update the `local_addrs` and `remote_addrs` in the respective `<IPSEC_MODE>/conf/<AUTH_MODE>/client.conf` file. Copy the client configuration from your repository:

```bash
sudo cp <IPSEC_MODE>/conf/<AUTH_MODE>/client.conf /etc/swanctl/swanctl.conf
# sudo cp <IPSEC_MODE>/conf/psk/secrets.conf /etc/ipsec.secrets
```

> [!TIP]
> `swanctl` completely ignores `/etc/ipsec.secrets`. It only reads secrets defined inside `/etc/swanctl/swanctl.conf`.

### 3. Start StrongSwan

```bash
# Start the daemon
sudo systemctl enable strongswan
sudo systemctl start strongswan
```

### 4. Load Configuration

```bash
sudo swanctl --load-all
```

> [!NOTE]
> If everything is configured correctly, the tunnel should establish automatically. If not, initiate it manually with `sudo swanctl --initiate --child tunnel-to-corp`.

## Verify IPsec Tunnel

### Check Tunnel Status

```bash
# On both server and client
sudo swanctl --list-sas
```

Output should show `ESTABLISHED` and `INSTALLED` status.

### Test Connectivity

```bash
# From client, ping server tunnel IP
ping -c 3 10.1.0.1

# Check tunnel traffic statistics
sudo swanctl --list-sas | grep -E "(bytes|packets)"
```

### Check IPsec Policies

```bash
# View IPsec policies
sudo ip xfrm policy list
sudo ip xfrm state list
```

### Monitor Logs

```bash
# View StrongSwan logs
sudo journalctl -u strongswan-swanctl -f

# Or check syslog
sudo tail -f /var/log/syslog | grep -E "(charon|ipsec)"
```

## Certificate Validation

### Verify Certificates

```bash
# Check certificate expiry
openssl x509 -enddate -noout -in /etc/swanctl/x509/server-cert.pem
openssl x509 -enddate -noout -in /etc/swanctl/x509/client-cert.pem

# Verify certificate chain
openssl verify -CAfile /etc/swanctl/x509ca/ca-cert.pem /etc/swanctl/x509/server-cert.pem
openssl verify -CAfile /etc/swanctl/x509ca/ca-cert.pem /etc/swanctl/x509/client-cert.pem

# View certificate details
openssl x509 -in /etc/swanctl/x509ca/ca-cert.pem -text -noout
```

## Troubleshooting

### Connection Issues

```bash
# Manually initiate tunnel
sudo swanctl --initiate --child tunnel-to-corp

# Reload configuration
sudo swanctl --load-all
```

If reloading configuration fails, try restarting strongswan:

```bash
sudo systemctl daemon-reload
sudo systemctl restart strongswan
sudo swanctl --load-all
```

### Network Debugging

```bash
# Check routing tables
ip route show

# Monitor with tcpdump
sudo tcpdump -i any -n port 500 or port 4500

# Test with verbose ping
ping -v -c 5 10.1.0.1
```

### Certificate Issues

```bash
# Check certificate validity
openssl x509 -in /etc/swanctl/x509/server-cert.pem -text -noout

# List supported algorithms
openssl list -signature-algorithms
```

## Cleanup

```bash
# Stop StrongSwan
sudo systemctl enable strongswan
sudo systemctl start strongswan

# Remove configuration
sudo rm -rf /etc/swanctl/*

# Remove tunnel interface (server only)
sudo ip link delete tunnel0
```
