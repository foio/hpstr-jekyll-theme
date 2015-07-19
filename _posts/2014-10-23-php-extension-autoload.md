---
layout: post
title: php扩展中autoload的实现
description: "php扩展中autoload的实现"
modified: 2014-10-23
tags: [hive hadoop]
image:
  background: triangular.png
comments: true
---

我们在写web应用程序时通常对每个类都建立一个 PHP 源文件。为了使用这些源文件，我们就需要在每个脚本开头写大量的的包含语句(include,require)。在 PHP 5 中，不再需要这样了。我们可__autoload()函数和spl_autoload_register函数实现实现自己的加载源文件的机制，它们会在试图使用尚未被定义的类时自动调用。通过调用这些函数，脚本引擎在 PHP 出错失败前有了最后一个机会加载所需的类。本文的主要目标是讲述如何在扩展中用C语言实现自动加载源文件的机制，但是在这之前我们先熟悉一下在PHP脚本中实现自动加载的方法。


###1. 在php脚本中实现自动加载

在 PHP 5 中我们可以定义一个 __autoload() 函数，它会在试图使用尚未被定义的类时自动调用，这样我们就可以定义一些自己的加载规则了。

{% highlight php %}
<?php
function __autoload($class_name) {
    require_once $class_name . '.php';
}

$obj  = new MyClass1();
$obj2 = new MyClass2();
?>
{% endhighlight %}

使用`spl_autoload_register`我们可以一次注册多个加载函数，PHP会在试图使用尚未被定义的类时按注册顺序调用。

{% highlight php %}
<?php
function autoload_services($class_name)
{
    $file = 'services/' . $class_name. '.php';
    if (file_exists($file))
    {
        require_once($file);
    }
}
function autoload_vos($class_name)
{
    $file = 'vos/' . $class_name. '.php';
    if (file_exists($file))
    {
        require_once($file);
    }
}
spl_autoload_register('autoload_services');
spl_autoload_register('autoload_vos');
?>

{% endhighlight %}


###2. 在php扩展中实现自动加载

最近在写一个php扩展，其中一个功能就是实现类的自动加载，其实也是通过在内核中调用`spl_autoload_register`函数来实现。使用zend API调用`spl_autoload_register`函数还是相对简单的，下面我们主要讲一下如何在内核中实现`inclue/require/include_once/require_once`等指令的功能。其实`inclue/require/include_once/require_once`等指令主要是读入文件编译并执行，下面的方法就是完成了这些操作，代码中有详细的注释。

####(1).在hive shell中加载

首先加载jar包，并创建临时函数\

{% highlight c %}
/*
*  loader_import首先将PHP源文件编译成op_array，然后依次执行op_array中的opcode
*/
int loader_import(char *path, int len TSRMLS_DC) {
    zend_file_handle file_handle;
    zend_op_array   *op_array;
    char realpath[MAXPATHLEN];

    if (!VCWD_REALPATH(path, realpath)) {
        return 0;
    }

    file_handle.filename = path;
    file_handle.free_filename = 0;
    file_handle.type = ZEND_HANDLE_FILENAME;
    file_handle.opened_path = NULL;
    file_handle.handle.fp = NULL;
    
    //调用zend API编译源文件
    op_array = zend_compile_file(&file_handle, ZEND_INCLUDE TSRMLS_CC);

    if (op_array && file_handle.handle.stream.handle) {
        int dummy = 1;

        if (!file_handle.opened_path) {
            file_handle.opened_path = path;
        }
        
        //将源文件注册到执行期间的全局变量(EG)的include_files列表中，这样就标记了源文件已经包含过了
        zend_hash_add(&EG(included_files), file_handle.opened_path, strlen(file_handle.opened_path)+1, (void *)&dummy,
                sizeof(int), NULL);
    }
    zend_destroy_file_handle(&file_handle TSRMLS_CC);

    //开始执行op_array
    if (op_array) {
        zval *result = NULL;
        //保存原来的执行环境，包括active_op_array,opline_ptr等
        zval ** __old_return_value_pp   =  EG(return_value_ptr_ptr);
        zend_op ** __old_opline_ptr     = EG(opline_ptr); 
        zend_op_array * __old_op_array  = EG(active_op_array);
        //保存环境完成后，初始化本次执行环境，替换op_array
        EG(return_value_ptr_ptr) = &result;
        EG(active_op_array)      = op_array;

#if ((PHP_MAJOR_VERSION == 5) && (PHP_MINOR_VERSION > 2)) || (PHP_MAJOR_VERSION > 5)
        if (!EG(active_symbol_table)) {
            zend_rebuild_symbol_table(TSRMLS_C);
        }
#endif
        //调用zend API执行源文件的op_array
        zend_execute(op_array TSRMLS_CC);
        //op_array执行完成后销毁，要不然就要内存泄露了，哈哈
        destroy_op_array(op_array TSRMLS_CC);
        efree(op_array);
        //通过检查执行期间的全局变量(EG)的exception是否被标记来确定是否有异常
        if (!EG(exception)) {
            if (EG(return_value_ptr_ptr) && *EG(return_value_ptr_ptr)) {
                zval_ptr_dtor(EG(return_value_ptr_ptr));
            }
        }
        //ok,执行到这里说明源文件的op_array已经执行完成了，我们要恢复原来的执行环境了
        EG(return_value_ptr_ptr) = __old_return_value_pp;
        EG(opline_ptr)           = __old_opline_ptr; 
        EG(active_op_array)      = __old_op_array; 

        return 1;
    }
    return 0;
}
{% endhighlight  %}


---

学习php扩展开发需要你对zend API以及PHP内核比较熟悉，推荐以下参考资料
 [php扩展入门博客](http://www.walu.cc/phpbook/index.md)
