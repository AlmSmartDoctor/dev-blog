---
layout: post
title: 리팩토링 후기
subtitle: Three Layer Architecture
gh-user: [calvin310486, HaTToek, hjjin0829]
gh-badge: [follow]
tags: [architecture]
comments: true
---

CRM팀에서는 원활한 CI/CD를 위해 3월 첫 주에 리팩토링과 관련한 이슈를 집중적으로 진행하는 리팩토링 기간을 가졌다.

프로젝트에 많은 변경 사항이 있었고, 그중 프로젝트 구조에 대해서 어떻게 변경했는지, 어떤 어려움이 있었는지 중점적으로 공유하고자 한다.

### Three Layer Architecture 

PL(Presentation Layer) : 유저와 상호작용을 한다. 

BLL(Business Logic Layer) : 모든 비즈니스 로직을 포함, Presentation Layer와 Data Access Layer가 직접적으로 상호작용하지 않게 한다.

DAL(Data Access Layer) : SQL server 같은 데이터에 접근한다.

{: .box-note}
Entity Layer : 3개의 계층에서 요청을 보내고 결과를 받을 수 있는 형식(인터페이스, 클래스)들을 담은 공통 Layer 이다.

장점

1. 코드가 각각 Layer에 분리되어 있어서 인터페이스, 비즈니스 로직, 쿼리 등 각각 단일 책임 원칙을 만족한다.
2. 유지보수 쉽다. 코드의 변경이 해당 Layer와 인접한 Layer에만 영향을 미친다.
3. 작업 분산이 용이하다. 팀원에 따라 할 일을 다른 계층으로 구성하여 각 팀원은 각 계층에 독립적으로 코드를 작성할 수 있으며, 
이는 개발자의 작업 부하를 제어하는 ​​데 도움이 된다.

### Three Layer simple data flow

1. Presentation Layer는 유저와 직접적으로 상호작용하는 유일한 Layer이다. 유저에게 데이터를 표현하거나, 
유저의 입력 데이터를 수집한다. 또한, 추가 처리를 위해 Business Logic Layer에 데이터를 전달한다.

2. Presentation Layer에서 데이터를 받은 Business Logic Layer에는 데이터 처리를 위한 로직이 담겨 있다.
이 계층에서는 데이터를 업데이트하거나 검색하는데, 직접적으로 데이터에 접근할 수 없기 때문에 Data Access Layer에 해당 사항들을 담은 request를 보낸다.

3. Data Access Layer는 Business Logic Layer로부터 request를 받아 쿼리를 제작하고 database에 이 요청을 실행한다. 
실행이 끝나면 그 결과를 담아 Business Logic Layer로 답변을 보낸다.

4. Business Logic Layer는 그 결과를 받고 로직을 완료하고 결과를 Presentation Layer로 보낸다.

5. Presentation Layer는 응답을 받고 유저에게 UI를 통해 표현한다.

![data flow](https://media.enlabsoftware.com/wp-content/uploads/2021/05/12224620/3Layer.jpg)

### CRM 프로젝트 구조 변경

Presenter는 왜 추가로 넣었는지? 왜 필요한지? 역할?

DAL위에 SystemCommon은 왜 최상위에 있는지?

프로젝트 구조를 위해 어떤 proj에서 뭘 지우고 참조를 어떻게 했는지?

무슨 프로젝트를 DAL, BLL, PL 중 어디로 보냈는지?

OLD폴더는 뭔지?

변경 전 프로젝트 구조 사진, 변경 후 사진 reshaper show project dependency diagram 로 추가?

Local Model, PL <-> BLL 을 할 때 Entity Layer 대신 다른 Layer 가 왜 필요한 지?

### CRM data flow example

CRM 코드와 함께 PL -> BLL -> DAL -> BLL -> PL 설명

### 참조

[How to build and deploy a three-Layer architecture application with C#](https://enlabsoftware.com/development/how-to-build-and-deploy-a-three-Layer-architecture-application-with-c-sharp-net-in-practice.html)