# Docker中的ENTRY POINT

`ENTRYPOINT`跟`CMD`有啥区别？

## ENTRYPOINT

允许用户将容器作为命令行运行。有2种运行模式：

### EXEC模式

```
ENTRYPOINT ["executable", "param1", "param2"]
```

EXEC模式用于设定容器运行时的默认运行命令。之后用户可使用CMD来传递额外的默认可变参数。EXEC模式不会启动SHELL来运行。

举个例子，`ENTRYPOINT ["echo","$HOME"]`中的`$HOME`不会被展开。

### Shell模式

```bash
ENTRYPOINT executable param1 param2
```

如果你给ENTRYPOINT传递一个字符串，那么它会使用Shell模式(`/bin/sh -c`)执行。这种模式会进行shell处理，环境变量展开等操作。**并且会忽略任何`CMD`或者`docker run`传递的命令行参数。**

## 差异和合作

`ENTRYPOINT`和`CMD`可以一起进行合作，遵循以下规范：

1. Dockerfile中必须至少指定一个`ENTRYPOINT`或者`CMD`
2. 当作为可执行程序运行时，必须指定`ENTRYPOINT`
3. `CMD`应该作为`ENTRYPOINT`的默认参数，如果将容器作为adhoc命令运行时。
4. `CMD`在运行时会被动态传递的参数覆盖

例子：

   None              |   NO ENTRYPOINT |    ENTRYPOINT a b | ENTRYPOINT ["a", "b"]
--------------|-----------------|---------------|----------------
No CMD     |    error, not allowed |   /bin/sh -c a b |  a b
CMD ["c", "d"] | c d | /bin/sh -c a b | a b c d 
CMD c d | /bin/sh -c c d | /bin/sh -c a b | a b /bin/sh -c c d 