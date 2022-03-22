---
layout: post
title:  "Submodule로 사용하던 공통 모듈을 Nuget Package로 이관하기"
date:   2022-03-21 15:25:29 +0900
categories: jekyll update
author: calvin310486
---

기존 사내에서 사용하던 Git Submodule 공통 모듈을 C#에서 제공하는 Nuget Package로 이관했던 경험을 나누고자 한다.

이관하는 과정을 순서대로 설명했다.

### Github Nuget Package 설정

Nuget 서버는 Github를 선택했다. Github Package를 사용하기 위해서는 개인 token을 Github에서 생성해야 한다.

```
Token 생성 경로: Github / Settings / Developer settings / Personal access tokens / Generate new token
```

![Untitled](/assets/images/github_packages/github_packages_22.png)

token의 scope는 write:packages와 read:packages를 선택한다.

![Untitled](/assets/images/github_packages/github_packages_23.png)

해당 화면의 token을 복사해둔다.

다음으로 token을 사용해 Nuget.config 파일을 작성한다.

```
파일 경로: %AppData%\\Nuget\\Nuget.config
```

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
    <packageSources>
        <add key="github" value="<http://nuget.pkg.github.com/OWNER/index.json>" />
    </packageSources>
    <packageSourceCredentials>
        <github>
            <add key="Username" value="MY USERNAME" />
            <add key="ClearTextPassword" value="MY TOKEN" />
        </github>
    </packageSourceCredentials>
</configuration>
```

- OWNER : Organization or github username
- Username : github username
- ClearTextPassword : Personal access token

특정 solution에만 Nuget.config를 적용하고 싶다면 .sln파일과 같은 폴더에 Nuget.config파일을 생성하면 된다.

사전 설정을 마치면 기존 git submodule에서 원하는 부분을 nuget package로 변경할 수 있다.

이번에는 모바일 파트에서 주로 사용하는 repository SmartMQ, SmartChartServer에서 공통으로 사용하는 SystemCommon/DataHandle/NetworkUtil.cs를 이관하기로 했다.

### Nuget Package 프로젝트 생성

먼저 새로운 nuget package를 위해 새로운 repository를 만들고 새 프로젝트를 생성한다.

package 이름은 프로젝트 이름을 따라가기 때문에 다른 사람이 만든 package와 겹치지 않게 이름을 정해야 한다.

![Untitled1](/assets/images/github_packages/github_packages_1.png)

밑줄 친 Newtonsoft.Json, Microsoft.Extensions.DependencyInjection들이 프로젝트 이름이자 패키지 이름이다.

만약 NetworkUtil.cs에서 이름을 따 프로젝트 이름을 Util로 하면 이름을 Util로 한 pakcage와 겹칠 수 있다.

따라서 프로젝트 이름을 SomeNamespace.Util과 같은 방식으로 차이를 두어 설정해야 한다.

![Untitled](/assets/images/github_packages/github_packages_3.png)

생성한 프로젝트를 Pack하기 위해서는 csproj 파일 설정이 필요하다.

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFrameworks>net35;net46</TargetFrameworks>
    <Version>1.0.0</Version>
    <RepositoryUrl>https://github.com/OWNER/YourPackageRepositoryName</RepositoryUrl>
  </PropertyGroup>

</Project>
```

해당 설정은 Package 생성을 위한 최소 설정이다. 이 외에도 Author, Owner 등 추가할 수 있는 태그가 있다.

### 기존 Git Submodule의 코드 복사

기존 SystemCommon/DataHandle/NetworkUtil.cs를 복사해서 SomeNamespace.Util 으로 복사한다.

추후 참조를 통해 사용할 것을 생각해 사용할 class, method를 public으로 열어두는 것 정도를 고려한다.

### Package 생성 및 배포

Visual Studio: Release 설정 후 프로젝트 우클릭 - pack

![Untitled](/assets/images/github_packages/github_packages_8.png)

SomeNamespace.Util.csproj가 있는 폴더에서

```powershell
dotnet pack --configuration Release
```

두 가지 방법 모두 좋다.

![Untitled](/assets/images/github_packages/github_packages_9.png)

프로젝트 이름(SomeNamespace.Util) + 버전(1.0.0) + .nupkg 로 생성된 것을 알 수 있다.

해당 파일을 배포한다.

```powershell
dotnet nuget push .\SmartDoctor.Util.1.0.0.nupkg --source "github"
```
—source “github” 는 처음 설정한 nuget.config 파일의 "github"이다.

해당 과정은 github action을 통해 자동화 가능하다.

[A NuGet package workflow using GitHub Actions](https://acraven.medium.com/a-nuget-package-workflow-using-github-actions-7da8c6557863)

### Package 다운로드

도구 - Nuget 패키지 관리자 

![Untitled](/assets/images/github_packages/github_packages_13.png)

![Untitled](/assets/images/github_packages/github_packages_14.png)

처음 Nuget.config에서 설정한 ”github”이다.

패키지 소스를 github 혹은 모두로 바꿔주면 배포한 SomeNamespace.Util을 볼 수 있다.

![Untitled](/assets/images/github_packages/github_packages_16.png)

![Untitled](/assets/images/github_packages/github_packages_17.png)

### 기존 코드 삭제 및 새 코드 추가

기존 코드를 삭제하고 사용하고 싶은 프로젝트에 새로 만든 SomeNamespace.Util 참조 추가 후 달라진 namespace에 맞춰 수정하면 된다.

여러 공통 모듈에 대해서 이관 과정을 거친 경험에 따르면 다음과 같은 과정이 편이하다.

1. 기존 Git Submodule의 코드 삭제
2. 새로운 Nuget Package 참조 추가
3. Namespace 수정
4. 추가 코드 다듬기 및 수정
5. 모듈을 사용했던 repository에 대해 1-4 반복


### 참조

[About Git Submodule](https://sgc109.github.io/2020/07/16/git-submodule/)

[About Github Package](https://musma.github.io/2019/09/30/github-package-registry.html)

[Newtonsoft.Json/Src/Newtonsoft.Json at master · JamesNK/Newtonsoft.Json](https://github.com/JamesNK/Newtonsoft.Json/tree/master/Src/Newtonsoft.Json)

[Working with the NuGet registry - GitHub Docs](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-nuget-registry)

[GitHub Package 사용하기: NuGet](https://musma.github.io/2020/12/17/github-package-registry-nuget.html)

[Overview of Hosting Your Own NuGet Feeds](https://docs.microsoft.com/en-us/nuget/hosting-packages/overview)

