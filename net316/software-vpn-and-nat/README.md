# Software VPN and NAT:

- A Virtual Private Network (VPN) connection allows you to extend private network over public network. In AWS,[Software VPN](https://docs.aws.amazon.com/aws-technical-content/latest/aws-vpc-connectivity-options/software-vpn-amazon-to-amazon.html){:target="_blank"} refers to VPN connection using a software VPN appliance running in your Amazon VPC network.

- Network Address Translator (NAT) translates one IP address into another by modifying network address in the IP header of packets in transit through a routing device.

## Objective:

- In this session we will use [Software VPN](https://docs.aws.amazon.com/aws-technical-content/latest/aws-vpc-connectivity-options/software-vpn-amazon-to-amazon.html){:target="_blank"} and NAT to connect to VPCs with overlapping IP address

![](images/softwareVpn_nat.png)

### Create two VPCs with overlapping IP address:

1. Create two VPCs:
   - NET316: VPC A
   - NET316: VPC B
2. Create public subnet in each VPC:
   - Public subnet is required to create IPsec VPN between two software VPN appliance running in AWS
3. Create custom route table in each VPC and associate subnet created in step(2) to this newly created custom route table
4. Create Amazon Linux 2 EC2 instance in each VPC:
   - These EC2 instances will have an overlapping IP address
5. Create IPsec VPN between two EC2 instances running AWS VPCs: NET316: VPC A and NET316: VPC B:
   - Install libreswan on both EC2 instances
   - Configure IPsec on both EC2 instances
   - Install iptables
   - Configure iptables to do NAT for overlapping IP address

### Verify:

- Verify you can ping from one EC2 instance to another
- Verify traffic is encapsulated

### Understand concepts:

- Software VPN and NAT
- Overlapping IP adddress
- How software VPN and NAT allows to connect two VPCs with overlapping ip address

## Workshop setup:

### Create ssh key-pair:

- **Skip this step** if you want to use your existing ssh key-pair

#### Using Amazon EC2 console

- To create ssh key-pair using Amazon EC2 Console, click on [Amazon EC2 console](https://eu-west-1.console.aws.amazon.com/ec2/).
  - Documentation can be found [here](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html#having-ec2-create-your-key-pair)
  - Verify region is set to eu-west-1 (Ireland)

### Configure AWS CLI:

- For this session will use AWS CLI to configure required AWS resources

#### Install:

- If you don't have AWS CLI installed or don't have up to date version, follow installion/upgrade instruction found [here](){:target="_blank"}


#### Configure:

- Verify installation:
```
aws version
```

- Configure AWS CLI:
```
aws configure
```
```
$ aws configure
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE --> This should be your IAM User access key ID. IAM User that you will use to configure AWS resources in this document
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY --> This should be your IAM User access key ID. IAM User that you will use to configure AWS resources in this document
Default region name [None]: eu-west-1 --> Region of your choice, for this session we will use eu-west-1 (Ireland) region
Default output format [None]: json --> format of your choice
```

### Create underlying infrastructure using AWS CloudFormation:

- Launch [Software VPN and NAT AWS CloudFormation Template](){:target="_blank"} to create:
  - 2 VPC: NET316: VPC A and NET316: VPC B
  - 2 public subnets, one in each VPC
  - 2 route tables, one in each VPC
  - 2 security groups, one in each VPC
  - 2 EC2 instances, one in each VPC

### Configure IPsec and iptabls on NET316: VPC A EC2 instance:

- configuratio needs to be performed as root:
```
sudo su
```

- Do yum update:
```
yum update all -y
```

- Configure IP forwarding:
```
```

- Install IP tables:
```
yum install iptables-services
yum info iptables-services
systemctl restart iptables
```

- Verify iptable installation:
```
iptables -L
iptables -L -t nat
```

- Install libreswan:
```
yum install libreswan -y
yum info libreswan
```

- Remove old database if any and re-initalize the database:
  - for libreswan there is no db entry
```
systemctl stop ipsec
rm /etc/ipsec.d/*db
ipsec initnss
systemctl stop ipsec
```

- Generate random key for ipsec secrets config, same key will be used on both the instances:
```
openssl rand -base64 48
```

- Configure IPsec:
  - From libreswan perspective left = local side of the VPN connection and right = remote side of the VPC connection
```
$: cat /etc/ipsec.d/vpcA-to-vpcB.conf
conn vpcA-to-vpcB
  authby=secret
  type=tunnel
  esp=aes_gcm128-null
  mtu=1436
  left=10.0.0.11
  leftid=52.212.20.244
  leftsubnet=192.168.1.0/24
  right=54.154.85.125
  rightsubnet=192.168.2.0/24
  auto=start
  dpdaction=restart
  dpddelay=10
  dpdtimeout=60
$: cat /etc/ipsec.d/vpcA-to-vpcB.secrets
52.212.20.244 54.154.85.125 : PSK "HVMWmcf1x/y51cz7VH56yjUVE0wwkH45ON1rgAb7awcr2r009e6DebmIep1rTc+r"
$:
```

- Configure iptables:
  - We will use an unused address space for translation.
    - For NET316: VPC A, we will use 192.168.1.0/24 as the unused address space.
    - For NET316: VPC B, we will use 192.168.2.0/24 as the unused address space.
  - 10.0.0.11 is the overlapping private IP address of NET316: VPC A EC2 instance and NET316: VPC A EC2 instance
```
iptables -t nat -A PREROUTING -d 192.168.1.11/32 -j DNAT —to-destination 10.0.0.11
[root@ip-10-0-0-11 ec2-user]# iptables -t nat -A POSTROUTING -s 10.0.0.11/32 -d 192.168.2.0/24 -j SNAT —to-source 192.168.1.11
```

### Configure IPsec and iptabls on NET316: VPC B EC2 instance:

- configuratio needs to be performed as root:
```
sudo su
```

- Do yum update:
```
yum update all -y
```

- Configure IP forwarding:
```
```

- Install IP tables:
```
yum install iptables-services
yum info iptables-services
systemctl restart iptables
```

- Verify iptable installation:
```
iptables -L
iptables -L -t nat
```

- Install libreswan:
```
yum install libreswan -y
yum info libreswan
```

- Remove old database if any and re-initalize the database:
  - for libreswan there is no db entry
```
systemctl stop ipsec
rm /etc/ipsec.d/*db
ipsec initnss
systemctl stop ipsec
```

- We will use same random key that was created above for ipsec secrets config

- Configure IPsec:
  - From libreswan perspective left = local side of the VPN connection and right = remote side of the VPC connection
```
$: cat /etc/ipsec.d/vpcB-to-vpcA.conf
conn vpcB-to-vpcA
  authby=secret
  type=tunnel
  esp=aes_gcm128-null
  mtu=1436
  left=10.0.0.11
  leftid=54.154.85.125
  leftsubnet=192.168.2.0/24
  right=52.212.20.244
  rightsubnet=192.168.1.0/24
  auto=start
  dpdaction=restart
  dpddelay=10
  dpdtimeout=60
$: cat /etc/ipsec.d/vpcB-to-vpcA.secrets
54.154.85.125 52.212.20.244 : PSK "HVMWmcf1x/y51cz7VH56yjUVE0wwkH45ON1rgAb7awcr2r009e6DebmIep1rTc+r"
$:
```

- Configure iptables:
  - We will use an unused address space for translation.
    - For NET316: VPC A, we will use 192.168.1.0/24 as the unused address space.
    - For NET316: VPC B, we will use 192.168.2.0/24 as the unused address space.
  - 10.0.0.11 is the overlapping private IP address of NET316: VPC A EC2 instance and NET316: VPC A EC2 instance
```
iptables -t nat -A PREROUTING -d 192.168.2.11/32 -j DNAT —to-destination 10.0.0.11
[root@ip-10-0-0-11 ec2-user]# iptables -t nat -A POSTROUTING -s 10.0.0.11/32 -d 192.168.1.0/24 -j SNAT —to-source 192.168.2.11
```

### Verification: NET316: VPC A EC2 Instance

- IPsec:
```
ipsec verify
ipsec status
ipsec show
ipsec whack --trafficstatus
```

- iptables:
```
iptables -L
iptables -L -t nat
```

### Verification: NET316: VPC B EC2 Instance

- IPsec:
```
ipsec verify
ipsec status
ipsec show
ipsec whack --trafficstatus
```

- iptables:
```
iptables -L
iptables -L -t nat
```

### Traffic test:

- From NET316: VPC A EC2 instance ping NET316: VPC B EC2 instance using the unused IP address:

- On NET316: VPC A EC2 Instance:
```
$: ping 192.168.2.11 -c 1
PING 192.168.2.11 (192.168.2.11) 56(84) bytes of data.
64 bytes from 192.168.2.11: icmp_seq=1 ttl=255 time=0.580 ms

--- 192.168.2.11 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.580/0.580/0.580/0.000 ms
$:
```

- On NET316: VPC A EC2 Instance, capture packets:
```
$: tcpdump -ni eth0 not port 22
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes

21:25:44.462716 IP 52.212.20.244.ipsec-nat-t > 10.0.0.11.ipsec-nat-t: UDP-encap: ESP(spi=0x49142d8b,seq=0x55), length 120
21:25:44.462775 IP 192.168.1.11 > 192.168.2.11: ICMP echo request, id 28479, seq 1, length 64
21:25:44.462812 IP 10.0.0.11.ipsec-nat-t > 52.212.20.244.ipsec-nat-t: UDP-encap: ESP(spi=0x2541e038,seq=0x52), length 120
21:25:56.077767 IP 52.212.20.244.isakmp > 10.0.0.11.isakmp: isakmp: phase 1 I ident
21:25:56.078107 IP 10.0.0.11.isakmp > 52.212.20.244.isakmp: isakmp: phase 1 R ident
21:25:56.079538 IP 52.212.20.244.isakmp > 10.0.0.11.isakmp: isakmp: phase 1 I ident
21:25:56.080591 IP 10.0.0.11.isakmp > 52.212.20.244.isakmp: isakmp: phase 1 R ident
21:25:56.083444 IP 52.212.20.244.ipsec-nat-t > 10.0.0.11.ipsec-nat-t: NONESP-encap: isakmp: phase 1 I ident[E]
21:25:56.083708 IP 10.0.0.11.ipsec-nat-t > 52.212.20.244.ipsec-nat-t: NONESP-encap: isakmp: phase 1 R ident[E]

21:26:06.094019 IP 10.0.0.11.ipsec-nat-t > 52.212.20.244.ipsec-nat-t: NONESP-encap: isakmp: phase 2/others R inf[E]
21:26:06.094635 IP 52.212.20.244.ipsec-nat-t > 10.0.0.11.ipsec-nat-t: NONESP-encap: isakmp: phase 2/others I inf[E]
21:26:06.094664 IP 52.212.20.244.ipsec-nat-t > 10.0.0.11.ipsec-nat-t: NONESP-encap: isakmp: phase 2/others I inf[E]
21:26:06.094822 IP 10.0.0.11.ipsec-nat-t > 52.212.20.244.ipsec-nat-t: NONESP-encap: isakmp: phase 2/others R inf[E]

^C
13 packets captured
13 packets received by filter
0 packets dropped by kernel
$:
```

### Clean up (delete AWS resources):

1. Delete [Software VPN and NAT AWS CloudFormation Template](){:target="_blank"}


## Considerations:

- Works with overlapping ip address
- EC2 instance in either VPC can intiate the connection. Allows bi-directional communication
- Mesh access. Many services between VPCs that need to communicate, which have some coordinated control or trust
- Scales to thousands of consumers
- Support TCP and UDP
- Requires a software VPN appliance
  - It your responsibility to manage software VPN appliance
- Can be complex to setup and requires IPsec VPN and NAT
- Traffic is encrypted but is not private. It is traversing over the internet


