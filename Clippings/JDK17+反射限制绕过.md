---
title: JDK17+反射限制绕过
source: https://pankas.top/2023/12/05/jdk17-%E5%8F%8D%E5%B0%84%E9%99%90%E5%88%B6%E7%BB%95%E8%BF%87/
created: 2025-08-28
description: 本文详细介绍了JDK17及更高版本对Java核心API反射的强封装限制，并通过修改调用类的module属性使其与`Object`类的module一致，利用`sun.misc.Unsafe`类成功绕过这些限制的实践方法。
finished: "false"
tag: clipper
cover: https://www.loliapi.com/acg/pc/
updated: 2025-09-27 22:34:06
---
