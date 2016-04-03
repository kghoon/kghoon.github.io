---
layout: post
title:  "EC2 인스턴스 프로비저닝을 위한 Chef #2"
categories: chef
comments: true
---

AWS EC2 인스턴스를 [Test kitchen][1]으로 사용하는 Chef Cookbook 개발 환경을 설정하는 과정을 설명한다. 

개발 도구 설치
---
Chef cookbook을 작성하고 테스트하는데 필요한 도구들을 설치 해 보자. Chef는 사용자의 Workstation에서 개발하는데 필요한 도구들을 모아 [Chef Development Kit][1] 이라는 이름의 패키지 형태로 제공한다.

[Chef Developement Kit Download][2] 페이지에서 사용자 OS에 맞는 버전을 선택하여 설치하면 된다. 윈도우 환경이라면 Powershell 또는 윈도우즈용 GIT을 설치하여 bash 환경을 구성하도록 한다.

EC2 Kitchen을 위한 Gem설치
---

작성한 Cookbook을 테스트할 수 있는 Sandbox로 [Test Kitchen][3]이라는 환경을 제공한다. 주로 Vagrant를 사용하여 Virtualbox VM에서 테스트하는 형태이지만, EC2를 Test kitchen으로 사용할 수도 있다.

Amazon Linux를 사용한다면 Vagrant보다는 EC2 환경에서 직접 테스트하는 것이 나아 보인다.

EC2를  Test kitchen으로 사용하려면 먼저 필요한 Ruby Gem을 설치해야 한다.

{% highlight bash %}
% chef gem install kitchen-ec2
% chef gem install chef-zero-scheduled-task
{% endhighlight %}

Cookbook 생성하기
---
`chef` 유틸리티를 사용하여 cookbook 템플릿을 하나 생성한다. 아래와 같이 출력되면 정상적으로 끝난 것이다. 

{% highlight bash %}
% chef generate cookbook java-webapp

Compiling Cookbooks...
Recipe: code_generator::cookbook
  * directory[/Users/jhkang/works/java-webapp] action create
    - create new directory /Users/jhkang/works/java-webapp
...
..
  * cookbook_file[/Users/jhkang/works/java-webapp/.gitignore] action create
    - create new file /Users/jhkang/works/java-webapp/.gitignore
    - update content in file /Users/jhkang/works/java-webapp/.gitignore from none to dd37b2
    (diff output suppressed by config)
{% endhighlight %}

AWS Credential 설정
---

### 1. IAM User 생성하기

Test kitchen은 EC2를 생성하고, 또 삭제하며, 테스트 할 때마다 Cookbook 코드를 인스턴스 내에서 실행한다. 이런 작업을 하려면 위 동작에 대한 권한을 가져야 하고, 그런 권한은 IAM에 유저를 생성하여 부여할 수 있다.

IAM User 생성 과정은 [아마존 웹 서비스를 다루는 기술 16장 - 2. IAM 사용자 생성하기][5] 에 자세하게 나와 있다.

위 내용에 따라 IAM 사용자를 생성하고, EC2에 대한 FullAccess를 부여하도록 한다.
또한 Access Key ID와 Secret Access Key를 포함하고 있는 파일을 다운로드 받아서 잘 보관한다.

### 2. $HOME/.aws/credentials 생성

IAM User 생성 후 다운로드 된 CSV 파일을 열면 Access Key ID와 Secret Access Key가 있는 것을 볼수 있다. 해당 내용을 아래와 같이 `$HOME/.aws/credentials` 에 적어준다.

{% highlight bash %}
[default]
aws_access_key_id=<your-aws-access-key-id>
aws_secret_access_key=<your-aws-secret-access-key>
{% endhighlight %}

Test kitchen 설정
---

`chef` 유틸리티로 Cookbook을 생성하면 Kitchen configuration 파일인 `{COOKBOOK_ROOT}/.kitchen.yml` 이 기본적으로 함께 생성된다.

아래의 내용을 참고하여 .kitchen.yml을 수정한다.

{% highlight yaml %}
---
driver:
  name: ec2
  aws_ssh_key_id: test-kitchen <= AWS EC2에 연결 시 사용할 Key pair 이름
  region: ap-northeast-2 
  availability_zone: a 
  subnet_id: subnet-da7593b3 <= 서브넷 아이디 (AZ에 맞는 것으로 설정)
  instance_type: t2.micro <= 인스턴스 타입. free tier에서 가능한 것은 't2.micro'
  image_id: ami-4d1fd123 <= Amazon linux의 AMI id. 
  security_group_ids: sg-d51be0bc <= 보안 그룹 아이디
  retryable_tries: 120

provisioner:
  name: chef_zero_scheduled_task

transport:
  ssh_key: C:\Users\jhkang\.ssh\test-kitchen.pem <= public key가 위치한 절대 경로
  username: ec2-user

platforms:
  - name: amazon

suites:
  - name: default
    run_list:
      - recipe[java-webapp::default]
    attributes:
{% endhighlight %}

Test kitchen 실행하기
---

아래와 같이 `kitchen create`를 실행하면 Test kitchen 생성된다. 아래와 같은 로그가 출력되면 성공적으로 생성 된 것이다.

{% highlight bash %}
% kitchen create
-----> Starting Kitchen (v1.4.2)
-----> Creating <default-amazon>...
       If you are not using an account that qualifies under the AWS
free-tier, you may be charged to run these suites. The charge
should be minimal, but neither Test Kitchen nor its maintainers
are responsible for your incurred costs.

...
..

       [SSH] Established
       Finished creating <default-amazon> (1m7.66s).
-----> Kitchen is finished. (1m8.32s)
{% endhighlight %}

`kitchen login` 을 실행하면 생성된 Test kitchen에 login할 수 있다.

{% highlight bash %}
% kitchen login
Last login: Mon Mar  7 15:20:05 2016 from 220.121.176.169

       __|  __|_  )
       _|  (     /   Amazon Linux AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-ami/2015.09-release-notes/
No packages needed for security; 3 packages available
Run "sudo yum update" to apply all updates.
[ec2-user@ip-172-31-3-157 ~]$
{% endhighlight %}

`kitchen destroy` 를 실행하면 생성된 Test kitchen EC2 instance가 삭제된다.

{% highlight bash %}
% kitchen destroy
-----> Starting Kitchen (v1.4.2)
-----> Destroying <default-amazon>...
       EC2 instance <i-9423a033> destroyed.
       Finished destroying <default-amazon> (0m1.02s).
-----> Kitchen is finished. (0m1.54s)
{% endhighlight %}

참고자료
---

- Chef Tutorials - [Install the virtualization tools](https://learn.chef.io/local-development/windows/get-set-up/get-set-up-ec2/)

[1]:https://docs.chef.io/workstation.html#chef-dk-title
[2]:https://downloads.chef.io/chef-dk/
[3]:https://docs.chef.io/kitchen.html
[4]:https://learn.chef.io/local-development/windows/get-set-up/get-set-up-ec2/
[5]:http://pyrasis.com/book/TheArtOfAmazonWebServices/Chapter16/02

 
