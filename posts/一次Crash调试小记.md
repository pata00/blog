# 一次Crash调试小记

一个类的成员函数，启动时调用程序直接core了，反复调试大半天到最后把函数体注释成空了调用依然崩溃，怀疑是其他线程改了这个对象，用gdb加内存断点没触发又crash了。

想想自己一直用最新版本的编译器，现在都已经(gcc version 11.1.0了) 难道是触发了编译器bug? 干脆再看下汇编好了，不看不知道，一看吓一跳。。。

这个函数执行完后居然不会跳转回去，继续执行了其他成员方法，果然是编译器bug！！！

咦，这个函数声明是bool的，漏写return了。。。

难道，非void类型函数，漏写return -O2优化下会真的没有return。。。

补上return true。一切果然恢复正常。。。

啊，我要记下2021年12月17日这个浪费了一天时间的问题。。。

是的，又浪费了一天生命。。。。。。。


# 大字报贴这里：

-   问题代码
    ```C
    bool MonitorManager::do_delay_tasks()
    {
        printf("call this function\n");
    }
    ```

- 生成的汇编指令
    ```assembly
    0000000000077b60 <_ZN14MonitorManager14do_delay_tasksEv>:
    77b60:   48 83 ec 08             sub    $0x8,%rsp
    77b64:   48 8d 3d 24 e8 30 00    lea    0x30e824(%rip),%rdi        # 38638f <_IO_stdin_used+0x138f>
    77b6b:   e8 90 e6 fd ff          call   56200 <puts@plt>  //没有多余参数的printf调用会被编译器优化成puts调用，这里调用puts函数后，后续没有任何返回指令,直接穿越到下一个成员函数体了。。。

    0000000000077b70 <_ZN14MonitorManager21ParseNameVersionIndexERKNSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEERS5_S8_Ri>:
    77b70:   41 57                   push   %r15   
    77b72:   48 8d 3d bf d4 30 00    lea    0x30d4bf(%rip),%rdi        # 385038 <_IO_stdin_used+0x38>
    77b79:   41 56                   push   %r14   
    77b7b:   41 55                   push   %r13   
    77b7d:   49 89 cd                mov    %rcx,%r13
    77b80:   41 54                   push   %r12   
    77b82:   4d 89 c4                mov    %r8,%r12
    77b85:   55                      push   %rbp   
    ```

- 补上return true的正确代码
    ```C
    bool MonitorManager::do_delay_tasks()
    {
        printf("call this function\n");
        return true;
    }
    ```

- 生成的汇编指令
    ```
    0000000000077b60 <_ZN14MonitorManager14do_delay_tasksEv>:
    77b60:   48 83 ec 08             sub    $0x8,%rsp
    77b64:   48 8d 3d 24 e8 30 00    lea    0x30e824(%rip),%rdi        # 38638f <_IO_stdin_used+0x138f>
    77b6b:   e8 90 e6 fd ff          call   56200 <puts@plt>
    77b70:   b8 01 00 00 00          mov    $0x1,%eax
    77b75:   48 83 c4 08             add    $0x8,%rsp
    77b79:   c3                      ret    
    77b7a:   66 0f 1f 44 00 00       nopw   0x0(%rax,%rax,1)

    0000000000077b80 <_ZN14MonitorManager21ParseNameVersionIndexERKNSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEERS5_S8_Ri>:
    77b80:   41 57                   push   %r15   
    77b82:   48 8d 3d af d4 30 00    lea    0x30d4af(%rip),%rdi        # 385038 <_IO_stdin_used+0x38>
    77b89:   41 56                   push   %r14   
    77b8b:   41 55                   push   %r13   
    77b8d:   49 89 cd                mov    %rcx,%r13
    77b90:   41 54                   push   %r12   
    77b92:   4d 89 c4                mov    %r8,%r12
    77b95:   55                      push   %rbp   
    ```