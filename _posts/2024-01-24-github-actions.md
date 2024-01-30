---
title: Analysis of Github Actions
subtitle: enforcing CICD immutability of your workflows
layout: post
author: martoc
image: https://martoc.github.io/blog/images/github.webp
---

The examination of GitHub Actions involves a comprehensive evaluation of its functionalities, features, and overall effectiveness. GitHub Actions is a powerful tool for automating workflows within the GitHub platform, enabling seamless integration and continuous delivery processes. It facilitates the automation of tasks such as code compilation, testing, and deployment, contributing to an efficient and streamlined development pipeline.

In conducting an analysis, it is essential to assess the various components and capabilities that GitHub Actions offers. This includes examining workflow configuration, understanding the use of actions, and evaluating the integration with version control repositories. Additionally, considerations should be given to factors such as scalability, flexibility, and ease of use.

Furthermore, the analysis should delve into the benefits and potential drawbacks associated with GitHub Actions. Assessing its impact on development speed, code quality, and collaboration among team members provides valuable insights into its real-world applicability.

In summary, a thorough analysis of GitHub Actions involves exploring its features, understanding its configuration, and evaluating its impact on development processes. This ensures a comprehensive understanding of its capabilities and aids in making informed decisions regarding its adoption in software development workflows.

GitHub offers three distinct methods for creating actions: 
JavaScript, compose, and container. 
Each of them possesses its own advantages 
and disadvantages, and, on the whole, 
all of them enforce mutability in your runs.

## JavaScript

This model installs Node.js on the running container. 
It is natively supported by the action engine, 
and therefore, GitHub manages the installation process. 
One positive aspect is that the JavaScript code can be 
packaged into a single file, eliminating the need to 
install dependencies.

An example action can be found at [martoc/my-custom-javascript-action](https://github.com/martoc/my-custom-javascript-action)

This action is used at [martoc/github-action-test](https://github.com/martoc/github-action-test/actions)

```yaml
name: javascript-version
run-name: ${{ github.event.head_commit.message }}
on: [push]
jobs:
  action:
    runs-on: ubuntu-22.04
    steps:
      - name: Run my javascript custom action
        uses: martoc/my-custom-javascript-action@3709a7f133773d77e83b4e151b77b142a73ce0a4
```

If you look at the logs for this execution, it looks like the nodejs is installed on the running container.

```
Run martoc/my-custom-javascript-action@3709a7f133773d77e83b4e151b77b142a73ce0a4
Mocking an AWS call via JS AWS SDK
```

Futher investigation shows that the nodejs is installed on the running container.

```bash
Run node --version || true
v18.19.0
```

However the version that the command requires is nodejs v20, if you run the same command on the container you will see that the version is v18.19.0
before and after the custom command is executed. Therefore the GitHub action engine for Javascript commands is installing nodejs on the running container or executing with nodejs v18.19.0. To check this I have added a command to the action to print the nodejs version.

```javscript
console.log('Node.js version: ' + process.version);
```

```bash
Run martoc/my-custom-javascript-action@799f06db8e843cbd3fc38b04fa3f6e4d19d615dd
Mocking an AWS call via JS AWS SDK
Node.js version: v20.8.1
```

## Compose

This configuration enables you to execute any shell script, 
including Python. However, Python must be installed in the 
running container, along with the dependencies required 
by your script.

## Container 

In this mode, the execution is wrapped in a container, 
offering a more controlled environment for 
running your scripts. However, this container is built 
during runtime.

# Conclusion

To maximize immutability during runtime, considering 
the mentioned options, the 
Container method provides a more 
controlled environment. However, 
it's essential to be aware that it 
still enforces mutability. 
To achieve maximum immutability, 
it's advisable to explore additional 
strategies or tools that specifically 
focus on immutable runtime environments 
for example Distributing binaries, where 
dependencies are fully contained and run directly 
in the running container, will diminish 
mutability to an HTTP pull.



