## docker pull 이 되지 않는 경우
 - 실습할 때, anonmyous로 사용하면 공인 IP를 기준으로 counting 하고, login을 하면 계정 사용자 기준으로 pull을 Counting 한다.
 ```bash
 [root@k8s-master ~]# yum install -y epel-release
 [root@k8s-master ~]# yum install -y jq
 ```
 * 로그인 하지 않았을 때 -anonymous 사용자
 ```bash  
 [root@k8s-master ~]# TOKEN_ANONY=$(curl "https://auth.docker.io/token?service=registry.docker.io&scope=repository:ratelimitpreview/test:pull" | jq -r .token)
```
 * 로그인했을 때 - 각자 계정(username:passwd 부분에 docker hub 계정 입력)
```bash
  [root@k8s-master ~]# TOKEN_USER=$(curl --user 'academyitwill:<password>' "https://auth.docker.io/token?service=registry.docker.io&scope=repository:ratelimitpreview/test:pull" | jq -r .token)
```
* 남은 rate-limit 수 확인 요청(익명 사용자) - 6시간에 100회 충전
```bash
[root@k8s-master ~]# curl --head -H "Authorization: Bearer $TOKEN_ANONY" https://registry-1.docker.io/v2/ratelimitpreview/test/manifests/latest
```
* 남은 rate-limit 수 확인 요청(계정 사용자) - 6시간에 200회 충전
```bash
[root@k8s-master ~]# curl --head -H "Authorization: Bearer $TOKEN_USER" https://registry-1.docker.io/v2/ratelimitpreview/test/manifests/latest
```
