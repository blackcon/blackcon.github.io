# 1\. Intro

github에서는 `August 13, 2021`부로 패스워드 인증 방식을 제거한다고 발표하였습니다. ([공지 링크](https://github.blog/2020-12-15-token-authentication-requirements-for-git-operations/)) 이후부터 원격지(ssh)에서 git push 및 기타 작업을 할 때 권한이 없다는 에러가 발생하고 있는데요. 개발자에게는 아주 중요한 부분이라, 이러한 에러를 어떻게 해결할 수 있을지 step-by-step으로 정리를 해보았습니다.

### 에러 요약

```
  fatal: Authentication failed for  'https://github.com/___/_____.git/'
```

### 에러 전체

```
  blackcon.github.io git:(master) ✗ git push origin master
  Username for 'https://github.com': 
  Password for 'https://blackcon@github.com':
  remote: Support for password authentication was removed on August 13, 2021.
  remote: Please see https://docs.github.com/en/get-started/getting-started-with-git/about-remote-repositories#cloning-with-https-urls for information on currently recommended modes of authentication.
  fatal: Authentication failed for 'https://github.com/___/_____.git/'
```

# 2\. Summary

위 에러를 해결하기 위한 세 줄 요약. 아래와 같습니다. 3줄 요약만으로 무슨 말인지 모르느 좀 더 상세한 설명을 이어서 다루어 볼게요.

> 1.  Personal Access Token 발급
> 2.  git clone 시 Personal Access Token을 사용하여 clone하기
> 3.  또는 git push 할 때 password가 아닌 Personal Access Tocken 사용하기

# 3\. Detail

거두절미하고 아래의 방법으로 설정을 해주신다면 ssh상으로 git을 제어할 수 있으니 따라해보시길 바랍니다.

### 1) Personal Access Token 발급

-   github 페이지에서 `Settings`를 클릭  
    [##_Image|kage@xZsOI/btrJt4MjllZ/tl5QoNokEO1XghoS4gd6ak/img.png|CDM|1.3|{"originWidth":588,"originHeight":1008,"style":"alignCenter"}_##]
-   Settings 페이지에 왔다면 좌측 하단에서 `Developer Settings` 클릭  
    [##_Image|kage@INl7u/btrJrQ8QRZb/N12p9ugwklbYkYkJKcMG80/img.png|CDM|1.3|{"originWidth":1754,"originHeight":1348,"style":"alignCenter"}_##]
-   `Personal Access Token` 클릭  
    [##_Image|kage@pwAvW/btrJu4yheXX/Nl3eJwqMsGvR2T9YzVeZPK/img.png|CDM|1.3|{"originWidth":2422,"originHeight":722,"style":"alignCenter"}_##]
-   `Generate New Token` 버튼을 클릭하여 새로운 토큰 설정하는 페이지로 이동[##_Image|kage@dVsvxw/btrJvQ7mEe7/RrDafxDXxse0E19aX65pa1/img.png|CDM|1.3|{"originWidth":1772,"originHeight":478,"style":"alignCenter"}_##]
-   해당 토큰에 부여할 권한을 체크한 후, 페이지 하단에 있는 `Generate Token` 클릭  
    [##_Image|kage@xcTuw/btrJwtD5xQs/gjslOPF8j2ZzQhUGrLl8vK/img.png|CDM|1.3|{"originWidth":1750,"originHeight":1080,"style":"alignCenter"}_##]  
    [##_Image|kage@yEFds/btrJwtYn8BD/iyAdPuMDfYdhwrzBRfekxK/img.png|CDM|1.3|{"originWidth":1684,"originHeight":498,"style":"alignCenter"}_##]

### 2) git clone 시 Personal Access Token을 사용하여 clone하기

-   Access key를 사용하여 clone하기
-   이 후 작업들은 인증절차 없이 바로 진행시킬 수 있는 장점이 있음
-   명령어
-   `$ git clone https://blackcon:{Personal_Token}@github.com/blackcon/blackcon.github.io.git Cloning into 'blackcon.github.io'... remote: Enumerating objects: 2932, done. remote: Counting objects: 100% (17/17), done. remote: Compressing objects: 100% (16/16), done. remote: Total 2932 (delta 6), reused 1 (delta 1), pack-reused 2915 Receiving objects: 100% (2932/2932), 33.21 MiB | 18.67 MiB/s, done. Resolving deltas: 100% (931/931), done.`

### 3) 또는 git push 할 때 password가 아닌 Personal Access Tocken 사용하기

-   위 절차가 아닌 과거 하던대로 git clone을 해두었거나,
-   매번 Personal token을 입력하면서 작업을 하고싶을 경우에는 아래 방법을 이용하면 됩니다.
-   명령어
-   `$ git clone https://github.com/user-or-organisation/myrepo.git Username: <my-username> Password: <my-personal-access-token> $ git push https://github.com/user-or-organisation/myrepo.git Username: <my-username> Password: <my-personal-access-token>`

# 3\. EOD (End of Document)

이상 개발중에 삽질을 하다가 발견한 이슈였습니다.. 동일한 증상이 있을 많은 개발자분들을 위해 포스팅을 하여보았고, 좀 더 새롭고 안전한 git life를 즐겨보도록 하겠습니다.
