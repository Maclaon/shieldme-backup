---
title: 基于阿里Java开发规范的CodeReview工具的设计
date: 2017-02-18 10:00:00
tags: CodeReview,
author: maclaon
comments: false
---
# 目标
> 工具的“强制性”保证了制度的执行，工具的“便捷性”最大程度减轻了工程师执行制度的负担，二者相辅相成。

一直以为工程师是神一般的人物，它们无所不能，它们能够利用各种方法，各种机制去解决各种问题，这也是我经常看"越狱"的因为，因为主角是一个有“神经病”的人，在精神病史上，这类人被称为天才。

在这里通过对现有代码的编译去规范代码审查，防止每次代码审查之后，下次又会出现同样的问题，导致代码审查完全没有效力，只是在浪费大家的时间，旨在通过工具的代码审查，让初步出现的问题，能够在人工代码审查之前进行修复，提高代码审查的执行和效率的提升。

## Java开发规范