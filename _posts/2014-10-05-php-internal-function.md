---
layout: post
title: php内核的自定义函数执行过程
description: "php内核的用户定义函数执行过程"
modified: 2014-10-05
tags: [php]
image:
  background: triangular.png
comments: true
---

在php内核中，用户定义函数使用下面的结构体表示：

{% highlight c %}
typedef union _zend_function {
    zend_uchar type;    
    struct {
        zend_uchar type;  /* never used */
        char *function_name;    //函数名称
        zend_class_entry *scope; //函数所在的类作用域
        zend_uint fn_flags;     // 作为方法时的访问类型等，如ZEND_ACC_STATIC等  
        union _zend_function *prototype; //函数原型
        zend_uint num_args;     //参数数目
        zend_uint required_num_args; //需要的参数数目
        zend_arg_info *arg_info;  //参数信息指针
        zend_bool pass_rest_by_reference;
        unsigned char return_reference;  //返回值 
    } common; 
    zend_op_array op_array;   //函数中的操作
    zend_internal_function internal_function;  
} zend_function;
{% endhighlight  %}

在结构体_zend_function中，最重要的字段就是op_array了，一个函数的执行过程，就是顺序地执行该opcode数组中的opcode。每一个opcode对应一组C函数。opcode和C函数的对应关系在zend_vm_execute.h的`zend_vm_get_opcode_handler`中设置，

{% highlight c %}
static opcode_handler_t zend_vm_get_opcode_handler(zend_uchar opcode, zend_op* op)
{
        static const int zend_vm_decode[] = {
            _UNUSED_CODE, /* 0              */
            _CONST_CODE,  /* 1 = IS_CONST   */
            _TMP_CODE,    /* 2 = IS_TMP_VAR */
            _UNUSED_CODE, /* 3              */
            _VAR_CODE,    /* 4 = IS_VAR     */
            _UNUSED_CODE, /* 5              */
            _UNUSED_CODE, /* 6              */
            _UNUSED_CODE, /* 7              */
            _UNUSED_CODE, /* 8 = IS_UNUSED  */
            _UNUSED_CODE, /* 9              */
            _UNUSED_CODE, /* 10             */
            _UNUSED_CODE, /* 11             */
            _UNUSED_CODE, /* 12             */
            _UNUSED_CODE, /* 13             */
            _UNUSED_CODE, /* 14             */
            _UNUSED_CODE, /* 15             */
            _CV_CODE      /* 16 = IS_CV     */
        };
        return zend_opcode_handlers[opcode * 25 
                + zend_vm_decode[op->op1.op_type] * 5 
                + zend_vm_decode[op->op2.op_type]];
}
{% endhighlight  %}

从`zend_vm_get_opcode_handler`中我们可以看出，操作码和操作数的类型共同决定了对应的C函数。每一个操作码最多对应25个C函数，php5.4的全部159个操作码对应的C函数在zend_vm_execute.h的的第36585行到第40560行。

了解了操作码数组相关知识后，让我们来看看php内核是如何执行一个用户定义函数的。

函数调用对应的C函数为ZEND_DO_FCALL_BY_NAME_SPEC_HANDLER，而最终的实现在zend_do_fcall_common_helper_SPEC中。zend_do_fcall_common_helper_SPEC主要对全局执行结构体_zend_executor_globals进行相应的设置，比如设置当前执行的操作码数组等，然后执行op_array。相关代码片段如下：

{% highlight c %}
EX(original_return_value) = EG(return_value_ptr_ptr);
EG(active_symbol_table) = NULL;
//设置_zend_executor_globals结构体的active_op_array为当前函数的op_array
EG(active_op_array) = &fbc->op_array;
EG(return_value_ptr_ptr) = NULL;
if (RETURN_VALUE_USED(opline)) {
    temp_variable *ret = &EX_T(opline->result.var);
    ret->var.ptr = NULL;
    EG(return_value_ptr_ptr) = &ret->var.ptr;
    ret->var.ptr_ptr = &ret->var.ptr;
    ret->var.fcall_returned_reference = (fbc->common.fn_flags & ZEND_ACC_RETURN_REFERENCE) != 0;
}
//如果已经设置zend_execute == execute这返回2，否则调用zend_execute执行当前函数的op_array
if (EXPECTED(zend_execute == execute)) {
    if (EXPECTED(EG(exception) == NULL)) {
        ZEND_VM_ENTER();
    } 
} else {
    zend_execute(EG(active_op_array) TSRMLS_CC);
} 
{% endhighlight  %}

执行op_array的execute函数的函数定义在zend_vm_execute.h中。execute首先为op_array对应的函数在内核函数调用栈中分配一个新的栈帧。然后初始化相关的执行数据。

{% highlight c %}
zend_vm_enter:
    //为当前函数分配栈帧
    execute_data = (zend_execute_data *)zend_vm_stack_alloc(
        ZEND_MM_ALIGNED_SIZE(sizeof(zend_execute_data)) +
        ZEND_MM_ALIGNED_SIZE(sizeof(zval**) * op_array->last_var * (EG(active_symbol_table) ? 1 : 2)) +
        ZEND_MM_ALIGNED_SIZE(sizeof(temp_variable)) * op_array->T TSRMLS_CC);
	 //初始化执行数据
    EX(CVs) = (zval***)((char*)execute_data + ZEND_MM_ALIGNED_SIZE(sizeof(zend_execute_data)));
    memset(EX(CVs), 0, sizeof(zval**) * op_array->last_var);
    EX(Ts) = (temp_variable *)(((char*)EX(CVs)) + ZEND_MM_ALIGNED_SIZE(sizeof(zval**) * op_array->last_var *
(EG(active_symbol_table) ? 1 : 2)));
    EX(fbc) = NULL;
    EX(called_scope) = NULL;
    EX(object) = NULL;
    EX(old_error_reporting) = NULL;
    EX(op_array) = op_array;
    EX(symbol_table) = EG(active_symbol_table);
    EX(prev_execute_data) = EG(current_execute_data);
    EG(current_execute_data) = execute_data;
    EX(nested) = nested;
{% endhighlight  %}

完成上述初始化工作后，将在一个while循环中执行当前op_array的每一个opcode。

{% highlight c %}
    while (1) {
        int ret;
#ifdef ZEND_WIN32
        if (EG(timed_out)) {
            zend_timeout(0);
        }
#endif
        if ((ret = OPLINE->handler(execute_data TSRMLS_CC)) > 0) {
            switch (ret) {
                case 1:
                    EG(in_execution) = original_in_execution;
                    return;
                case 2:
                    op_array = EG(active_op_array);
                    goto zend_vm_enter;
                case 3:
                    execute_data = EG(current_execute_data);
                default:
                    break;
            }
        }

    }
{% endhighlight  %}

opcode通过`OPLINE->handler(execute_data TSRMLS_CC)`执行，一个opcode的执行结果有四种状态，根据四种状态分别执行不同的流程。这里我们主要看状态为2的情况，opcode的执行结构状态为2表示该opcode是一个函数调用操作码，这种情况下，首先替换当前的op_array，`op_array = EG(active_op_array)`，然后跳转到zend_vm_enter处，为新函数分配堆栈并初始化执行环境。

{% highlight c %}
#define ZEND_VM_CONTINUE()         return 0
#define ZEND_VM_RETURN()           return 1
#define ZEND_VM_ENTER()            return 2
#define ZEND_VM_LEAVE()            return 3
{% endhighlight  %}

我们在execute函数中看到了为函数调用分配堆栈相关的操作，但是为什么没有函数调用完毕后的释放堆栈相关的操作呢？

php用户定义函数，无论是否有返回值，其对应的op_array都会有RETURN操作码，而RETURN操作码就是负责释放堆栈的。具体的实现在zend_vm_execute.h文件的`zend_leave_helper_SPEC`函数中,。

{% highlight c %}
zend_vm_stack_free(execute_data TSRMLS_CC);
{% endhighlight  %}






