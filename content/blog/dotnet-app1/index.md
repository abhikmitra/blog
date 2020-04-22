---
title: Running Dot Net Core outside MS ecosystem - 1
date: "2020-04-13T18:00:00.284Z"
description: "Create and run an ASP.net app"
---

### Create a Repo 
1. Using github for creating my repo , even though Github is a part of Microsoft , it still does not fall into my typical day to day flow .
2. As of writing, sadly I was not able to get a gitgnore in the list of github .gitignore so we will initialize the repo with just a readme
3. You can find the repository [here](https://github.com/abhikmitra/dotnet-outside-ms)

### Create an asp.net core app

1. Clone the app

```
git clone git@github.com:abhikmitra/dotnet-outside-ms.git
```
2. I will be using Visual Studio 2019 as my preferred editor

3. Our stack will be ASP.NET Core 3.1 on .NET Core so that it is cross platform. I will start of with a Simple API template.

4. I am not selecting the the "Enable Docker Setup" as I want to do that myself. I have named the solution `BackendApp` and the project is `BackendAPI`
5. Since Github doe snot have a .gitignore template for  dot net apps, we are gonna use this [.gitignore](https://raw.githubusercontent.com/OmniSharp/generator-aspnet/master/templates/gitignore.txt). This might not be enough , we will see.

6. The app is going to expose one endpoint called `api/environment/iscontainerized`. The controller checks for the environment variable "containerized" and returns if its true or false. If you run the app , it should return false.

End State: [The repo at the end of Part 1](https://github.com/abhikmitra/dotnet-outside-ms/tree/1CreateApp)  