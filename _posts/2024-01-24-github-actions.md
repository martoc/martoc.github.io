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



