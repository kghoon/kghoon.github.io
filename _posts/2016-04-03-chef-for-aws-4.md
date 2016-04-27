---
layout: post
title:  "Chef for AWS - Tomcat 설치 및 App deploy"
categories: chef
comments: true
---

EC2 인스턴스에 Tomcat을 설치하고 Sample application을 deploy하는 과정을 설명한다.

Tomcat 설치
-----

지난 시간에 이어서, Tomcat 8을 설치하고 Sample application을 sysvinit service 형태로 실행하는 과정을 수행해 보려고 한다.

먼저 마찬가지로 Chef supermarket에서 tomcat cookbook을 찾아본다.
Tomcat cookbook은 Chef software에서 직접 관리하고 있다.

가이드를 살펴보면 `tomcat_install`, `tomcat_service` 2개의 resource를 제공하고 있다.

우선, tomcat cookbook에 대한 dependency를 `metadata.rb`에 아래와 같이 적어준다.

{% highlight ruby %}
depends 'java', '~> 1.39.0'
depends 'tomcat', '~> 2.0.3'
{% endhighlight %}

`recipe/default.rb` 파일에 `tomcat_install` resource를 사용하여 tomcat을 설치할 수 있도록 아래 내용을 추가한다.

tomcat cookbook에서 tomcat_install resource 호출 시 옆에 명시하는 parameter는 instance name으로 사용된다. 이 instance name을 사용하여 user/group을 생성하고 sysvinit service로 등록될 때 service 이름으로도 사용되니 어플리케이션을 잘 나타내는 이름을 사용하는 것이 좋다.

{% highlight ruby %}
tomcat_install 'my-app' do
  version '8.0.32'
end
{% endhighlight %}

Test kitchen에 적용하여 잘 설치되는지 확인한다.

{% highlight bash %}
$ kitchen converge
...
       [2016-04-03T08:20:30+00:00] INFO: Report handlers complete
       Chef Client finished, 0/15 resources updated in 04 seconds
       Finished converging <default-amazon> (0m14.40s).
-----> Kitchen is finished. (0m14.87s)
$ kitchen login

[ec2-user@ip-172-31-13-130 ~]$ ls -l /opt
합계 12
drwxr-xr-x 5 root          root 4096  2월 10 18:03 aws
drwxr-xr-x 4 root          root 4096  4월  3 08:16 chef
lrwxrwxrwx 1 root          root   26  4월  3 08:18 tomcat_my-app -> /opt/tomcat_tomcat_8_0_32/
drwxr-x--- 9 tomcat_tomcat root 4096  4월  3 08:18 tomcat_tomcat_8_0_32
[ec2-user@ip-172-31-13-130 ~]$
{% endhighlight %}

Tomcat service 등록
-----

배포에 사용할 Sample application은 Tomcat에서 포함 된 Sample application이다.
WAR의 URL은 아래와 같다.

`https://tomcat.apache.org/tomcat-8.0-doc/appdev/sample/sample.war`

위 과정에 설치 된 tomcat의 webapps 디렉토리의 경로는 `/opt/tomcat_my-app/webapps` 이다.

WAR를 webapps 디렉토리에 ROOT.war로 다운로드 받는 것 부터 해보겠다.
http를 통해 remote파일을 내려받아 사용할때는 remote_file이라는 builtin resource를 사용하게 된다.

업데이트를 고려하여 기존에 `webapps/ROOT` directory가 있는 경우 삭제하고 service를 재시작할 수 있도록 notification하는 코드도 함께 넣어 준다.

아래와 같이 `recipe/default.rb` 에 코드를 추가하면 된다.

{% highlight ruby %}

# remove old application
directory "/opt/tomcat_my-app/webapps/ROOT" do
  action :delete
  recursive true
  notifies :restart, 'tomcat_service[my-app]'
end

remote_file "/opt/tomcat_my-app/webapps/ROOT.war" do
    source "https://tomcat.apache.org/tomcat-8.0-doc/appdev/sample/sample.war"
    user "tomcat_my-app"
    group "tomcat_my-app"
end
{% endhighlight %}

이제 sysvinit service를 등록하는 과정이다. 따로 customize 해야 할 부분이 없어보이므로 아래와 같은 간단한 코드만 추가하면 된다.

{% highlight ruby %}
tomcat_service 'my-app' do
    action :start
end
{% endhighlight %}

이제 테스트 키친에 적용하여 잘 동작하는지 확인해 볼 차례다.

{% highlight bash %}
$ kitchen converge
...
$ kitchen login
$ curl localhost:8080

<html>
<head>
<title>Sample "Hello, World" Application</title>
</head>
<body bgcolor=white>

<table border="0">
<tr>
<td>
<img src="images/tomcat.gif">
</td>
<td>
<h1>Sample "Hello, World" Application</h1>
<p>This is the home page for a sample application used to illustrate the
source directory organization of a web application utilizing the principles
outlined in the Application Developer's Guide.
</td>
</tr>
</table>

<p>To prove that they work, you can execute either of the following links:
<ul>
<li>To a <a href="hello.jsp">JSP page</a>.
<li>To a <a href="hello">servlet</a>.
</ul>

</body>
</html>
{% endhighlight %}

