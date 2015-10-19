+++
date = "2015-10-19T23:38:27+03:00"
draft = false
title = "Simple continous integration using Github and CodeDeploy"
+++

This post describes the work-in-progress solution for continous integration of [Селото](https://selo.to). It uses only Github, AWS CodeDeploy and a few shell scripts specific to the deployed system's architecture.

Connecting Git and CodeDeploy is nicely explained in [the AWS blog](http://blogs.aws.amazon.com/application-management/post/Tx33XKAKURCCW83/Automatically-Deploy-from-GitHub-Using-AWS-CodeDeploy). After following the article a git push results in the latest code pushed to an EC2 instance. If this is enough for your use case (ex. a static website/static blog), you are all set!

If the code needs to be build you are in for more work. The last part of the AWS article suggests using a CI system to acheive this and points you to [a list of alternatives](http://aws.amazon.com/codedeploy/product-integrations/). None of them was convenient for me - either another system to maintain (ex. Jenkins) or another service to sign up for. The best option at the time of writing is AWS CodePipeline, which is unfortunately only available in the US East AWS region. [Селото](https://selo.to) serves customers in Bulgaria and US East is a long way from there.

So, let me show you how CodeDeploy can act as a CI system. [CodeDeploy](http://docs.aws.amazon.com/codedeploy/latest/userguide/app-spec-ref.html) supports executing scripts before and after the code is deployed to the EC2 host. We can bundle these scripts with the code (say in the *scripts/* directory) so that they get deployed to the host. Here is the *appspec.yml*:

```yml
version: 0.0
os: linux
files:
    - source: /
      destination: /tmp/codeDeploy-build
hooks:
  BeforeInstall:
    - location: scripts/clean-build-directory.sh
  AfterInstall:
    - location: scripts/build.sh
    - location: scripts/test.sh
    - location: scripts/deploy.sh
```

1. We specify that all files from the source revision will be copied to */tmp/codeDeploy-build* (during the Install hook).
2. In the BeforeInstall hook, we clean the build directory from the previous build (*/tmp/codeDeploy-build*). I like doing this before the build to make debugging failed builds easier as the code is still present in the directory.
3. In the AfterInstall hook the code is already deployed to /tmp/codeDeploy-build and we can execute any scripts to build, test and deploy the application depending on our architecture. The next blog post will examine how this works for [Селото](https://selo.to) - a Tomcat 8 web app running on [ECS](https://aws.amazon.com/ecs/details/).

Important to note is that the scripts should be made up of expressions chained with *AND* i.e:

```bash
echo 1 && \
echo 2
```
This ensures that if one of the expressions fails the script will return the appropriate error code and cause the deploy to fail. 

That's it - a simple CI using vanilla CodeDeploy. Another useful hook that CodeDeploy supports is the ValidateService hook which is the last step of deployment and can be used to sound an alarm if the newly deployed service is unhealthy.

I have not yet implemented notifications for successful/failed deployments. Thinking of wrapping all the shell scripts in one which sends the result of the build to an SNS topic. Let me know in the comments below if you figured it out.
