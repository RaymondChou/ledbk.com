---
title: ruby中字符串动态调用类/方法
date: '2013-03-29'
description:ruby中字符串动态调用类/方法
categories:ruby
tags:ruby
---

#### demo

	class_name = "JiJi"

	method_name = "do_get"

	class JiJi
		def do_get
			puts 'success'
		end
	end


	if(Kernel.const_defined?(class_name))
		classname = Kernel.const_get(class_name)
		classname.new().send(method_name)
	else
		p "未找到类"
	end


#### 关于Class, Module,Object,Kernel

Ruby里，可以直接写puts, print等，感觉像是命令动词一样，这和我们说的Ruby里一切都是对象有点冲突，其实我们理解了Ruby中Class, Module,Object,Kernel的关系就明白了

Object是Ruby中所有类的父类，Object混入了Kernel这个模块，所以Kernel中内建的核心函数就可以被Ruby中所有的类和对象访问。

Object的实例方法由Kernel模块定义。

我们可以把Kernel理解为系统预定义的一些方法，我们可以在所有的对象上使用，使用时不需要使用类型作为前缀，当然我们也可以加上Kernel

对于一个普通的对象，可以直接调用Kernel的public method

而要想调用一个普通对象所包含的Kernel的函数，用一般的调用方法无法做到，只有通过Send来实现.
