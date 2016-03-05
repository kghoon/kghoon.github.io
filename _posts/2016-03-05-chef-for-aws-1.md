---
layout: post
title:  "EC2 인스턴스 프로비저닝을 위한 Chef #1"
categories: chef aws
comments: true
---

코드 기반의 서버 프로비저닝을 위한 도구로서 가장 널리 사용되고 있는 Chef! 그렇지만 초기 학습 커브가 만만치 않은 이 Chef를 AWS에서 사용하기 위한 용도로 Focus를 맞추어서 살펴 보고자 한다.

# 용어 알아보기

용어들이 쉐프의 입장에서 필요한 재료들을 나열하는 메타포를 사용하고 있으므로 이에 맞추어 생각하면 더 이해하기가 편하다.

## Components

### Cookbook

- 쉐프에게 필요한 요리책 (사실 쉐프님들은 요리책이 필요 없으시겠지만..)
- 보통 하나의 목적을 가진다. 그 목적이 서버 인스턴스의 프로비저닝일 수도 있고, Redis나 Nginx와 같은 소프트웨어 스택일 수도 있다.
- Cookbook은 recipe와 resource그리고 attribute들로 구성된다.
- [Chef supermark](https://supermarket.chef.io) 이라는 repository가 있어서, 서로 공유할 수 있다.

### Recipe

- 쉐프님의 요리 레시피. Cookbook의 Goal을 달성하기 위한 일련의 절차를 기술하는 것이다.
- 기본 recipe의 이름은 'default', chef-client가 실행하는 단위가 recipe이고, `$ chef-client -r 'recipe[cookbook-name]'` 이렇게 cookbook 이름만 전달한 경우 default recipe가 실행된다.

### Resource

- 쉐프님이 요리를 위한 식재료
- 많이 사용되는 resource로는 file(파일 생성/삭제), directory(디렉토리 생성/삭제), package(필요 패키지 설치), service(sysvinit/systemd service의 등록/시작/중지 등)가 있다.
- Chef에서 기본적으로 제공되는 built-in resource와 사용자에 의해 작성된 custom resource로 나눌 수 있다.
- Resource와 Provider는 인터페이스와 구현의 관계이다.
- 보통 OS와 그 버전에 따라 수행해야 할 동작이 달라지는 경우에 하나의 'resource' 인터페이스와 여러 개의 'provider' 구현을 갖는다.
- 그럴 필요가 없는 경우엔 그냥 Resource가 구현까지 포함한다.
- Custom resource는 보통 cookbook의 resource와 provider폴더에 위치한다

### Attribute

- 포트 번호나 사용자 이름, 설치 경로과 같이 configurable한 것들을 따로 관리하는 것을 말함.
- attributes 디렉토리에 파일을 생성하고 `default[KEY-A][KEY-B] = VALUE` 형태로 작성한다.
- Runtime에는 default가 아닌 `node`라는 변수명으로 접근이 가능하고 `node.default[KEY-A][KEY-B] = NEW_VALUE` 와 같은 형태로 값을 Overwrite할 수 있다.

### Template
- Nginx site configuration 파일등과 같이 customize가 필요한 텍스트 형태의 파일.
- Ruby ERB 형태로 작성한다.

## 운영 모델

### Server-Client mode

프로비저닝할 대상이 될 서버 인스턴스들을 관리 할 Chef 서버가 따로 존재하고, 여기에 대상 서버 인스턴스들이 노드로 붙어 있는 형태이다.

서버는 Chef를 개발한 Opscode에서 Saas 형태로 제공하는 hosted chef server가 있고, 사용자가 직접 open source chef server를 설치해서 사용할 수도 있다.

Hosted Chef Server는 Node 5개 까지만 무료로 제공한다.

사용자는 Chef Server에 Cookbook을 업데이트하고, Chef Server에서 Node를 업데이트하는 식으로 운영된다.

### Local mode

별도의 Chef server를 운영하지 않고, 대상 서버 인스턴스에 Chef client만 설치해서 사용하는 형태를 말한다.

`chef-client` 실행시 `-z` option을 주면 현재 디렉토리의 cookbooks 디렉토리에서 적용할 cookbook의 recipe를 찾아 실행한다.

AWS의 Opsworks가 이 형태로 chef를 사용하고 있고, 보통 CloudFormation을 사용하여 AutoScalingGroup에서 운영되는 EC2 인스턴스들을 프로비저닝하는 경우에도 local mode를 사용한다.

# 참고 자료

- [Chef Documents](http://docs.chef.io)
- [Chef Tutorial](https://learn.chef.io/tutorials/)
- [AWS OpsWorks User Guide - Welcome](http://docs.aws.amazon.com/opsworks/latest/userguide/welcome.html)
- [AWS CloudFormation Sample Template - Use Chef to deploy WordPress](https://s3-ap-northeast-2.amazonaws.com/cloudformation-templates-ap-northeast-2/WordPress_Chef.template)

