# AWS Client VPN



Best-practice to securely give remote admins access to private VPC resources (EC2s, databases, etc.) without public exposure.


---

## Set Up Easy-RSA and Generate Certificates


### Step 1: Install Easy-RSA on Windows

1. **Download Easy-RSA:**
    
    - Go to: https://github.com/OpenVPN/easy-rsa/releases
    - Download the latest `EasyRSA-X.X.X-win64.zip`
2. **Extract the files:**
    
    ```cmd
    # Create a working directory
    mkdir C:\easy-rsa
    
    # Extract downloaded ZIP to C:\easy-rsa
    # Your folder structure should look like:
    # C:\easy-rsa\EasyRSA-X.X.X\
    ```



### Step 2: Initialize the PKI (Public Key Infrastructure)

```cmd
# Initialize PKI structure
EasyRSA-Start.bat
```

This opens a new shell. In the Easy-RSA shell, run:

```bash
# Initialize PKI
./easyrsa init-pki
```

### Step 3: Build Certificate Authority (CA)

```bash
# Create the CA certificate
./easyrsa build-ca nopass
```

When prompted:

- **Common Name**: Enter `VPN-CA` (or any name you prefer)
- Press Enter

Output:

```
CA creation complete and you may now import and sign cert requests.
Your new CA certificate file for publishing is at:
C:/easy-rsa/EasyRSA-X.X.X/pki/ca.crt
```

### Step 4: Fix Missing Index File

**If you're using Git Bash on Windows**, you need to create the missing index.txt file manually:

```bash
# Create the index.txt file
touch pki/index.txt

# Create the serial file
echo "01" > pki/serial
```



### Step 5: Generate Server Certificate and Key

```bash
# Generate server certificate without password
./easyrsa build-server-full server nopass
```


Output files created:

- `pki/ca.crt` - CA certificate
- `pki/issued/server.crt` - Server certificate
- `pki/private/server.key` - Server private key

### Step 6: Generate Client Certificate and Key

```bash
# Generate client certificate without password
./easyrsa build-client-full client1.domain.tld nopass
```


Output files created:

- `pki/issued/client1.domain.tld.crt` - Client certificate
- `pki/private/client1.domain.tld.key` - Client private key

### Step 7: Locate Your Certificate Files

Your certificates are in these Windows paths:

```
C:\easy-rsa\EasyRSA-X.X.X\pki\ca.crt                          (CA Certificate)
C:\easy-rsa\EasyRSA-X.X.X\pki\issued\server.crt              (Server Certificate)
C:\easy-rsa\EasyRSA-X.X.X\pki\private\server.key             (Server Private Key)
C:\easy-rsa\EasyRSA-X.X.X\pki\issued\client1.domain.tld.crt  (Client Certificate)
C:\easy-rsa\EasyRSA-X.X.X\pki\private\client1.domain.tld.key (Client Private Key)
```





## Upload Certificates to AWS Certificate Manager (ACM)


### Step 8: Upload Server Certificate to ACM

1. **Open AWS Console** → Navigate to **Certificate Manager** (ACM)
2. Click **Import a certificate**
3. Fill in the fields:
    - **Certificate body**: Copy contents of `server.crt`
    - **Certificate private key**: Copy contents of `server.key`
    - **Certificate chain**: Copy contents of `ca.crt`
4. **Note the ARN** of the imported certificate (you'll need this later)


### Step 9: Upload Client Certificate to ACM

1. In **Certificate Manager**, click **Import a certificate** again
2. Fill in the fields:
    - **Certificate body**: Copy contents of `client1.domain.tld.crt`
    - **Certificate private key**: Copy contents of `client1.domain.tld.key`
    - **Certificate chain**: Copy contents of `ca.crt`
3. **Note the ARN** of this certificate as well

---

## Create AWS Client VPN Endpoint

### Step 10: Create Client VPN Endpoint

1. Create a VPC with Private Subnets
2. **Create Client VPN Endpoint**


**Client IPv4 CIDR:**

- Enter: `10.0.0.0/22` (provides 1024 IP addresses)
- This should NOT overlap with your VPC CIDR

**Server certificate ARN:**

- Select the server certificate ARN you imported earlier

**Authentication Options:**

- Select: **Use mutual authentication**
- **Client certificate ARN**: Select the client certificate ARN

**Connection Logging:**

- Choose: **No** (or enable if you want CloudWatch logging)


**Transport Protocol:**

- Select: **UDP** (recommended for better performance)

**VPN Port:**

- Leave as: **443**

**Split Tunnel:**

- Select: **Enable** (only routes VPC traffic through VPN)

**VPC ID:**

- Select your target VPC

**Security Groups:**

- Select a security group that allows VPN traffic

**Self-service portal:**

- Select: **Disable** 


---

## Configure Network Associations and Routes

### Step 11: Associate Target Network (Subnets)

1. Click on your **Client VPN Endpoint**
2. Go to **Target network associations** tab
3. Click **Associate target network**
4. **VPC**: Your VPC is already selected
5. **Choose a subnet to associate**: Select a **private subnet** from your VPC
6. Click **Associate target network**
7. **Repeat** for additional subnets in different AZs for high availability
8. Go to **Authorization rules** tab
9. Click **Add authorization rule**
10. **Destination network**: Enter your VPC CIDR (e.g., `10.1.0.0/16`)
11. **Grant access to**: Select **Allow access to all users**
12. **Description**: `Allow access to VPC resources`
13. Click **Add authorization rule**

### Step 12: Configure Route Table (if needed)

1. Go to **Route table** tab
2. The VPC CIDR route is automatically added when you associate subnets
3. If you need to add custom routes:
    - Click **Create route**
    - **Route destination**: Enter CIDR (e.g., `10.2.0.0/16`)
    - **Target VPC Subnet ID**: Select associated subnet
    - Click **Create route**

---



## Configure Security Groups

### Step 13: Update Security Groups for Private Resources

For resources you want to access (EC2, RDS, etc.):

1. Go to **EC2** → **Security Groups**
2. Select the security group attached to your private resources
3. Click **Edit inbound rules**
4. **Add rules** for required access:
    - **SSH**: Type: SSH, Protocol: TCP, Port: 22, Source: `10.0.0.0/22` (VPN CIDR)
    - **RDP**: Type: RDP, Protocol: TCP, Port: 3389, Source: `10.0.0.0/22`
    - **Other services**: Add as needed
5. Click **Save rules**

---

## Download and Configure VPN Client

### Step 14: Download Client Configuration

1. Go back to **Client VPN Endpoints**
2. Select your endpoint
3. Click **Download client configuration**
4. Save the `.ovpn` file (e.g., `downloaded-client-config.ovpn`)

### Step 15: Modify Configuration File

1. Open the downloaded `.ovpn` file in a text editor
2. Add the following **at the end** of the file:

```
<cert>
[Paste contents of client1.domain.tld.crt here]
</cert>

<key>
[Paste contents of client1.domain.tld.key here]
</key>
```

3. Save the modified file

### Step 16: Install VPN Client Software


- Download **AWS VPN Client**: https://aws.amazon.com/vpn/client-vpn-download/
- Install the application

---


## Connect and Test

### Step 17: Connect to VPN

**Using AWS VPN Client**

1. Open **AWS VPN Client**
2. Click **File** → **Manage Profiles** → **Add Profile**
3. **Display Name**: `My VPC VPN`
4. **VPN Configuration File**: Browse and select your modified `.ovpn` file
5. Click **Add Profile**
6. Select the profile and click **Connect**



### Step 19: Test Access to Private Resources

- Create a Private EC2 

```bash
# SSH to private EC2 instance
ssh -i "kp.pem" ec2-user@30.0.139.93

# Test connectivity
ping 30.0.139.93
```

---



## Cost Considerations

- **Client VPN Endpoint**: ~$0.10/hour (per endpoint)
- **Connection Hours**: ~$0.05/hour (per active connection)
- **Data Transfer**: Standard AWS data transfer rates apply
