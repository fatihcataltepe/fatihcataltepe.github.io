---
layout: post
title: Gitignore file for sbt project in IntelliJ IDEA
date:   2017-01-15 23:14:24 +0300
categories: gitignore .gitignore sbt scala intellij idea template
---

It is very important to configure .gitignore file properly before starting a project.
Here, I am sharing my `.gitignore` file that I have been using for almost all projects and it is quite useful so far.

```
target
*.class
*.iml
*.ipr
logger
.idea
!.idea/workspace.xml

# File-based project format:
*.iws

# mpeltonen/sbt-idea plugin
.idea_modules/

# IntelliJ
/out/

# JIRA plugin
atlassian-ide-plugin.xml
```