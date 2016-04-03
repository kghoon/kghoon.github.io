---
layout: post
title:  "EC2 인스턴스 프로비저닝을 위한 Chef #3"
categories: chef aws
comments: true
---

EC2 인스턴스에 JDK를 설치하는 recipe를 작성한다.

설치 과정을 생각해보자
-----

Amazon linux가 설치 된 초기 상태의 EC2 인스턴스에 SSH로 로그인해서 Java 8로 작성 된 Web application을 실행할 수 있는 환경을 만든다면 어떤 과정들이 필요할지를 먼저 생각해보자.

아마도 대충 아래와 같은 과정이 필요할 것이다.

1. Java 8 runtime 설치
2. Tomcat v8.x 설치
3. 어플리케이션 구동을 위한 Tomcat 설정 작업
4. Sysvinit service로 등록
5. Nginx 또는 Apache 설치
6. Reverse proxy 설정을 위한 configuration 추가

위와 같은 작업을 chef를 통해 수행하더라도 같은 일을 마찬가지로 수행한다. 하지만 어플리케이션 서버 프로비저닝을 위해 누구나 하게되는 이러한 일련의 과정들을 불필요하게 반복하지 않도록 chef에서는 이런 일들을 수행하는 cookbook을 공유할 수 있는 [Chef Supermarket][1]을 제공하고 있다.

Java, Tomcat, Nginx cookbook을 사용하여 위와 같은 작업을 수행하는 recipe를  작성해 보도록 하겠다.

Chef supermarket에서 필요한 Cookbook 찾아보기
-----

Chef supermarket에서 java를 검색하면 여러개의 cookbook들이 검색된다. 신뢰할 만한 Cookbook을 찾는 기준은 보통 Most Followed가 된다.

이를 기준으로 아래와 같은 Cookbook들을 사용할 것이다.

{% highlight text %}

'java', '~> 1.39.0'
'tomcat', '~> 2.0.3'
'nginx', '~> 2.7.6'

{% endhighlight %}

Cookbook 작성 시작하기
-----

아래와 같이 `chef generate <cookbook-name>` 명령어를 실행하여 cookbook작성을 위한 skeleton을 생성할 수 있다.

{% highlight bash %}
$ chef generate my-app
Compiling Cookbooks...
Recipe: code_generator::cookbook
  * directory[/Users/jhkang/works/test/my-app] action create
    - create new directory /Users/jhkang/works/test/my-app
  * template[/Users/jhkang/works/test/my-app/metadata.rb] action create_if_missing
...
..
  * cookbook_file[/Users/jhkang/works/test/my-app/.gitignore] action create
    - create new file /Users/jhkang/works/test/my-app/.gitignore
    - update content in file /Users/jhkang/works/test/my-app/.gitignore from none to dd37b2
    (diff output suppressed by config)

$ ls 
my-app
{% endhighlight %}

Java8 설치
=====

내가 작성하는 것도 완전한 형태의 하나의 cookbook이고, 내가 사용하려고 하는 java나 nginx, tomcat cookbook들 역시 각각 별도의 cookbook이다. 

이렇게 다른 cookbook을 사용하려면 사용 할 cookbook의 이름과 버전 정보를 metadata.rb에 아래와 같이 적어준다.

{% highlight ruby %}
name 'my-app'
maintainer 'The Authors'
maintainer_email 'you@example.com'
license 'all_rights'
description 'Installs/Configures my-app'
long_description 'Installs/Configures my-app'
version '0.1.0'

depends 'java', '~> 1.39.0'
{% endhighlight %}

다음은 이 cookbook을 어떻게 사용해야 하는지를 찾아봐야 한다. Chef Supermark에서 [Java cookbook][2]을 찾아 들어가면 설명이 적혀 있다.

description을 보면 java cookbook의 경우 java cookbook의 default recipe를 include하기만 하면 설치가 되고, 기본적으로 openjdk 6가 설치된다고 되어 있다.

우리는 oracle jdk 8 설치 해볼 것이므로, 어떤 속성들을 변경해야 oracle jdk 8을 설치할 수 있는지 찾아봐야 한다.

찾아보니 아래와 같은 속성들을 변경하면 될 것 같다.

{% highlight ruby %}
node['java']['install_flavor'] = 'oracle'
node['java']['jdk_version'] = '8'
{% endhighlight %}

이제 이 내용을 내 코드에 적용해 보자

먼저 java default recipe를 include하는 코드를 `recipes/default.rb`에 아래와 같이 추가한다.

{% highlight ruby %}
include_recipe 'java'
{% endhighlight %}

그리고 변경해야할 속성들을 적용해야한다. recipe에 직접 위의 내용을 적어 줄 수도 있지만, 속성들을 담고 있는 코드는 구조적으로 별도로 분리할 수 있도록 되어 있다.

아래와 같이 `chef generate attritube <NAME>` 을 실행하여 attribute 파일을 생성한다.

{% highlight bash %}
$ chef generate attribute default
Compiling Cookbooks...
Recipe: code_generator::attribute
  * directory[/Users/jhkang/works/test/my-app/attributes] action create
    - create new directory /Users/jhkang/works/test/my-app/attributes
  * template[/Users/jhkang/works/test/my-app/attributes/default.rb] action create
    - create new file /Users/jhkang/works/test/my-app/attributes/default.rb
    - update content in file /Users/jhkang/works/test/my-app/attributes/default.rb from none to e3b0c4
    (diff output suppressed by config)
{% endhighlight %}

그후 `attributes/default.rb`에 아래와 같이 변경해야 할 속성을 적어 준다.

{% highlight ruby %}
default['java']['install_flavor'] = 'oracle'
default['java']['jdk_version'] = '8'
{% endhighlight %}

이제 여기까지 하면 JDK가 잘 설치가 되야할 것으로 예상된다. 이제 Test kitchen을 사용하여 실제 EC2 instance에 적용해 보도록 한다.

Test kitchen 설정 방법은 이전 [포스트][3]를 참고 하도록 한다.

{% highlight bash %}
$ kitchen converge
-----> Starting Kitchen (v1.4.2)
/opt/chefdk/embedded/lib/ruby/gems/2.1.0/gems/httpclient-2.6.0.1/lib/httpclient/webagent-cookie.rb:458: warning: already initialized constant HTTPClient::CookieManager
/opt/chefdk/embedded/lib/ruby/gems/2.1.0/gems/httpclient-2.6.0.1/lib/httpclient/cookie.rb:8: warning: previous definition of CookieManager was here
-----> Creating <default-amazon>...
       If you are not using an account that qualifies under the AWS
free-tier, you may be charged to run these suites. The charge
should be minimal, but neither Test Kitchen nor its maintainers
are responsible for your incurred costs.

       Instance <i-374eda90> requested.
       EC2 instance <i-374eda90> created.
       Waited 0/600s for instance <i-374eda90> to become ready.

...
[2016-04-03T06:44:11+00:00] FATAL: You must set the attribute node['java']['oracle']['accept_oracle_download_terms'] to true if you want to download directly from the oracle site!


================================================================================
Error executing action `install` on resource 'java_ark[jdk]'
================================================================================

SystemExit
----------
exit

{% endhighlight %}

에러가 발생했다. tarball url을 따로 명시하지 않고 oracle site에서 직접 다운로드 받으려면 `node['java']['oracle']['accept_oracle_download_terms']`를 `true`로 설정해야 한다는 내용이다.

해당 attribute를 `attributes/default.rb`에 추가한 후 다시 실행 해 보도록 한다.

{% highlight ruby %}
default['java']['install_flavor'] = 'oracle'
default['java']['jdk_version'] = '8'
default['java']['oracle']['accept_oracle_download_terms'] = true
{% endhighlight %}

{% highlight bash %}
$ kitchen converge
-----> Starting Kitchen (v1.4.2)
/opt/chefdk/embedded/lib/ruby/gems/2.1.0/gems/httpclient-2.6.0.1/lib/httpclient/webagent-cookie.rb:458: warning: already initialized constant HTTPClient::CookieManager
/opt/chefdk/embedded/lib/ruby/gems/2.1.0/gems/httpclient-2.6.0.1/lib/httpclient/cookie.rb:8: warning: previous definition of CookieManager was here
-----> Converging <default-amazon>...
       Preparing files for transfer
       Preparing dna.json
       Resolving cookbook dependencies with Berkshelf 4.0.1...
       Removing non-cookbook files before transfer
       Preparing validation.pem
       Preparing client.rb
...
..
[2016-04-03T06:53:23+00:00] INFO: Chef Run complete in 151.164355358 seconds
       Running handlers:
       [2016-04-03T06:53:23+00:00] INFO: Running report handlers
       Running handlers complete
       [2016-04-03T06:53:23+00:00] INFO: Report handlers complete
       Chef Client finished, 2/8 resources updated in 02 minutes 32 seconds
       Finished converging <default-amazon> (2m38.58s).
-----> Kitchen is finished. (2m39.04s)
{% endhighlight %}

정상적으로 실행이 끝났다. 로그인해서 제대로 설치가 되었는지 확인해 볼 수 있다.

{% highlight bash %}
$ kitchen login
/opt/chefdk/embedded/lib/ruby/gems/2.1.0/gems/httpclient-2.6.0.1/lib/httpclient/webagent-cookie.rb:458: warning: already initialized constant HTTPClient::CookieManager
/opt/chefdk/embedded/lib/ruby/gems/2.1.0/gems/httpclient-2.6.0.1/lib/httpclient/cookie.rb:8: warning: previous definition of CookieManager was here
Last login: Sun Apr  3 06:50:50 2016 from 220.121.176.169

       __|  __|_  )
       _|  (     /   Amazon Linux AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-ami/2015.09-release-notes/
8 package(s) needed for security, out of 33 available
Run "sudo yum update" to apply all updates.
Amazon Linux version 2016.03 is available.
[ec2-user@ip-172-31-15-82 ~]$ java -version
java version "1.8.0_65"
Java(TM) SE Runtime Environment (build 1.8.0_65-b17)
Java HotSpot(TM) 64-Bit Server VM (build 25.65-b01, mixed mode)
[ec2-user@ip-172-31-15-82 ~]$ logout
Connection to ec2-52-79-74-72.ap-northeast-2.compute.amazonaws.com closed.
{% endhighlight %}

다음 포스팅에서는 Tomcat을 설치하고 Sysvinit service로 등록하는 과정을 진행해 보도록 하겠다.


[1]:https://supermarket.chef.io
[2]:https://supermarket.chef.io/cookbooks/java
[3]:http://www.ghoon.net/chef/aws/2016/03/07/chef-for-aws-2.html

