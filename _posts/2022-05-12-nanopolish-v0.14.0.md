---
layout: post
title: Nanopolish v0.14.0 - M1 Support
author: richard
draft: false
comments: true
---

A new release of [nanopolish](https://github.com/jts/nanopolish) has been pushed to GitHub.  By popular demand, `nanopolish v0.14.0` now supports the M1 processor.  This new release requires the installation of a M1 native supported `gcc` from [`homebrew`](https://brew.sh) as the pre-installed `clang` compiler is not supported.  To build `nanopolish`:
```
make CC=gcc-11 CXX=g++-11 CPP=cpp-11 arm_neon=1
```

