---
type: artifact
category: about
status: active
author: Chora Studio
title: Research Artifacts
description: An brief introduction to research artifacts.
created: 2026-11-15
updated: 2026-11-15
---

# Research Artifacts

This web page is a research artifact but a research artifact is more than a simple web page. This web page is just one way of viewing and interacting with this particular research artifact. While options for this research artifact may be limited at this time, Chora Studio is exploring other ways of viewing and interacting with research artifacts and artifacts more broadly.

For now, and within the context of Chora Studio, a research artifact starts out as a markdown file. This research artifact started as a markdown file in a directory (`about`) inside another directory (`research`) inside another directory (`chora-studio`). In this particular setup, the parent directory (`chora-studio`) is a knowledge base for Chora Studio. The knowledge base is another topic so this research artifact will turn focus on the subdirectories. 

The more direct subdirectory (`research`) is also a Git repository, so changes to markdown files in the subdirectory (`about`) of the subdirectory (`research`), or any other subdirectory of the subdirectory, can be tracked using Git and pushed to a remote repository (`https://github.com/chora-studio/artifacts`). After changes are committed locally and pushed remotely, the changes are pulled into a Git submodule (`research`) in another repository (`website`).

The second repository (`website`) is the source code for a website (`https://chora.studio`) and it also has a remote repository (`https://github.com/chora-studio/website`). The first repository (`research`) is a submodule in the second repository (`website`), so when changes are pushed from the first (`research`), the submodule (`research`) in the second repository (`website`) needs to pull those changes before the artifacts can be published.

 The source code for the website is written in a way that all markdown files within the Git submodule (`research`) are compiled as web pages when the source code is compiled. The compiled source code is pushed to a branch on the remote repository that automatically deploys the website including all the compiled research artifacts in the form of web pages. The source code also compiles an [index page](../) with the list of research artifacts.

 So back from the start, the research artifacts are written as markdown files (a widely supported and portable format), changes to those files are tracked with Git and pushed to a remote repository (tracking changes are important for several reasons), and then those changes are pulled into a submodule in another repository and compiled and deployed as web pages.
