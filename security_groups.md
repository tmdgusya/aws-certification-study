# Security Groups

- Security Group 은 EC2 의 방화벽이다.
## Functions
- Port 로의 접근을 통제
- Authorised IP Ranges - IPv4 and IPv6
- Control of inbound network (from other to the instance)
- Control of outbound network (from the instance to other)
### Good to know
- Can be attached to multiple instances
- Lock down to a region / VPC combination
	- So, if we moved to other regions, then we need to create new security group
- Does live "outside" the ec2 - if traffic is blocked the EC2 instance won't see it
- It's good to maintain one separate security group for SSH access.
### Referencing other security groups 
- 이걸 하는 이유는 아래와 같은 상황으로 이해하면 좀 더 편한데
	- A EC2 instance has A Security Group (Name is ASG)
	- B EC2 instance has B Security Group (Name is BSG)
		- 이때 B 에서 A 로 네트워크 요청을 해야 한다면, A instance 의 security group 에 BSG 를 추가해주면 된다. 그렇게 하면 쉽게 B instance 로부터 trafiic 을 allow 할 수 있다.
		- IP 를 신경쓰지 않아도 되어서 정말 간편함.
### Connect EC2 instance by ssh 

command

```sh
ssh ec2-user@public-ip
```

위의 커맨드를 실행하면, Permission Denied 가 나게 되는데 그 이유는 우리가 생성한, Access Key 를 사용하지 않았기 때문이다.
그래서 우리가 만든 Access Key 를 커맨드에 포함해서 넣어줘야 한다.

```sh
ssh -i EC2Tutorial.pem ec2-user@public-ip
```

위와 같이 입력하게 되면 우리가 생성한 Access Key 를 통해서 접근한다. 하지만 첫 시도에는 반드시 아래와 같은 문구를 마주한다.

```sh
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@         WARNING: UNPROTECTED PRIVATE KEY FILE!          @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
```

이 이유는 파일을 처음 다운로드 하게 되면 권한이 0644 라는 public 한 권한을 가지고 있는데, 
이로 인해 private key 가 유출될 우려가 있어 공유되지 않도록 권한을 제한해야 한다.

```sh
chmod 0400 EC2Tutorial.pem
```
