---
layout: post
title:  "EC2 인스턴스 프로비저닝을 위한 Chef #5"
categories: chef aws
comments: true
---

EC2 인스턴스에 Nginx를 설치하고 Tomcat이 리스닝 중인 포트로 Reverse Proxy를 설정하는 방법을 설명한다.

Nginx 설치
-----

이번에도 Chef supermarket에서 nginx cookbook을 찾아 설치할 것이다. 
지금 기준으로 2.7.6이 최신 버전이다.

`metadata.rb`에 아래와 같이 dependency를 명시한다.

{% highlight ruby %}
depends 'nginx', '~> 2.7.6'
{% endhighlight %}

가이드 내용에 보면 Configuration 가능한 속성들의 종류는 많지만, 설치하는 방법은 크게 두가지 이다.
Package manager(아마도 yum)을 통해 설치하려면 그냥 nginx::default recipe를 include하면 되고, source를 직접 compile하여 설치하려면 nginx::source recipe를 include하면 된다.

설정들은 기본대로 두고, Package manager를 사용하여 nginx를 설치해보자.
간단히 아래와 같이 `recipes/default.rb`에 한 줄을 추가한다.

{% highlight ruby %}
include_recipe 'nginx'
{% endhighlight %}

Test kitchen에 적용하여 제대로 설치되는지 확인해 보도록 하자.

{% highlight bash %}

$ kitchen converge
...
$ kitchen login
$ curl localhost
<html>
<head><title>404 Not Found</title></head>
<body bgcolor="white">
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.8.1</center>
</body>
</html>
{% endhighlight %}

404에러가 나긴 하지만, 제대로 설치가 된 것을 확인했다.

Reverse Proxy 설정
---
이제 80번 포트로 오는 요청을 8080 포트를 리스닝 중인 톰캣으로 전달해주는 Proxy 설정을 Nginx에 추가해보자.

`chef generate template <FILENAME>` 을 실행하여 reverse proxy nginx configuration 설정을 작성할 template파일을 하나 생성한다.

{% highlight bash %}
$ chef generate template nginx-proxy.conf
Compiling Cookbooks...
Recipe: code_generator::template
  * directory[/Users/jhkang/works/test/my-app/templates/default] action create
    - create new directory /Users/jhkang/works/test/my-app/templates/default
  * template[/Users/jhkang/works/test/my-app/templates/default/nginx-proxy.conf.erb] action create
    - create new file /Users/jhkang/works/test/my-app/templates/default/nginx-proxy.conf.erb
    - update content in file /Users/jhkang/works/test/my-app/templates/default/nginx-proxy.conf.erb from none to e3b0c4
    (diff output suppressed by config)
{% endhighlight %}

Chef template 은 루비의 ERB(Embedded Ruby) syntax를 사용하여 작성한다.
Nginx proxy 설정을 위해 `templates/nginx-proxy.conf.erb`를 아래와 같이 작성한다.

{% highlight nginx %}
server {
    listen       80 <% if @default -%>default<% end -%>;
<% unless @server_name.nil? -%>
    server_name  <%= @server_name %>;
<% end -%>
    location / {
        proxy_set_header   X-Forwarded-Host         $host;
        proxy_set_header   X-Forwarded-Server       $host;
        proxy_set_header   X-Forwarded-For          $proxy_add_x_forwarded_for;

        proxy_pass http://127.0.0.1:<%= @proxied_port %>;
    }
} 
{% endhighlight %}

이 템플릿을 EC2에 업로드하기 위해 `recipes/default.rb` 에 아래와 같은 코드를 추가한다.

{% highlight ruby %}
template "#{node['nginx']['dir']}/sites-available/my-app" do
  source 'nginx-proxy.conf.erb'
  owner 'root'
  group 'root'
  mode '0644'
  variables(
    :proxied_port => 8080,
    :default => true,
  )
  action :create
end
{% endhighlight %}

nginx cookbook은 debian site enable/disable style을 따르고, 이를 지원하기 위해 nxensite, nxdissite script를 지원한다.

위 nxensite, nxdissite의 wrapper definition인 nginx_site를 사용하여 위에서 업로드한 configuration을 활성화한다.

{% highlight ruby %}
nginx_site 'my-app'
{% endhighlight %} 

이제 Test kitchen에 적용하여 잘 동작하는지 확인해 보도록 하자.

{% highlight bash %}
$ kitchen converge
...
$ kitchen login
...
$ curl localhost
<html>
<head>
<title>Sample "Hello, World" Application</title>
</head>
<body bgcolor=white>
...
</html>
[ec2-user@ip-172-31-5-3 ~]$
{% endhighlight %}

Test kitchen 생성 시 사용한 security group의 inbound 설정이 80번 포트를 허용하고 있다면, 웹브라우저를 통해서도 확인이 가능하다.

모든 작업이 끝났다. `recipes/default.rb` 의 전체 코드를 보면 다음과 같다.

{% highlight ruby %}
include_recipe 'java'

tomcat_install 'my-app' do
  version '8.0.32'
end

directory "/opt/tomcat_my-app/webapps/ROOT" do
    action :delete
    recursive true
    notifies :restart, 'tomcat_service[my-app]', :delayed
end

remote_file "/opt/tomcat_my-app/webapps/ROOT.war" do
    source "https://tomcat.apache.org/tomcat-8.0-doc/appdev/sample/sample.war"
    user "tomcat_my-app"
    group "tomcat_my-app"
end

tomcat_service 'my-app' do
    action :start
end

include_recipe 'nginx'

template "#{node['nginx']['dir']}/sites-available/my-app" do
    source 'nginx-proxy.conf.erb'
    owner 'root'
    group 'root'
    mode '0644'
    variables(
        :proxied_port => 8080,
        :default => true,
    )
    action :create
end

nginx_site 'my-app'
{% endhighlight %}

Refactor cookbook
---

작성 된 코드에서 리팩토링해야 할 만한 부분이 무엇인지 찾아보자.

- 'my-app' 이라는 instance 이름이 중복되어 작성되어 있다. 또한 설정 가능하게 하는 것이 좋을 것 같다.
- WAR URL의 경우도 설정 가능 하게 변경하는 것이 나아 보인다.

위의 사항들을 적용하기 위해 `attributes/default.rb` 에 아래의 속성들을 추가한다.

{% highlight ruby %}
default['my-app']['name'] = 'my-app'
default['my-app']['war'] = 'https://tomcat.apache.org/tomcat-8.0-doc/appdev/sample/sample.war'
{% endhighlight %}

수정 된 `recipes/default.rb`는 아래와 같다.

{% highlight ruby %}
include_recipe 'java'

app_name = node['my-app']['name']

tomcat_install app_name do
  version '8.0.32'
end

directory "/opt/tomcat_#{app_name}/webapps/ROOT" do
    action :delete
    recursive true
    notifies :restart, "tomcat_service[#{app_name}]", :delayed
end

remote_file "/opt/tomcat_#{app_name}/webapps/ROOT.war" do
    source node['my-app']['war']
    user "tomcat_#{app_name}"
end

tomcat_service app_name do
    action :start
end

include_recipe 'nginx'

template "#{node['nginx']['dir']}/sites-available/#{app_name}" do
    source 'nginx-proxy.conf.erb'
    owner 'root'
    group 'root'
    mode '0644'
    variables(
        :proxied_port => 8080,
        :default => true,
    )
    action :create
end

nginx_site app_name
{% endhighlight %}

마무리..
---

Chef를 사용하여 기본적인 java web application을 프로비저닝하는 과정을 살펴보았다. 이미 잘 작성되어 있는 신뢰할 만한 cookbook들을 활용하면 적은 양의 코드로 원하는 작업들을 손쉽게 해결 할 수 있는 매력이 있다.

다음 포스팅에서는 지금까지 작성한 cookbook을 Test kitchen이 아닌 Cloud Formation을 사용하여 프로비저닝하는 과정을 살펴 보려고 한다.


