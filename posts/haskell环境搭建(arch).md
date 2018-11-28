# haskell环境搭建(arch)

## 安装ghc和cabal
```sudo pacman -S ghc cabal-install```

## cabal的基本使用
- 先创建一个工程目录:

      mkdir progtest
      cd progtest
- 创建一个cabal工程

      cabal init

- 由于arch的ghc不带有boot静态库还需增加配置

      cabal new-configure --disable-library-vanilla --enable-shared --enable-executable-dynamic
- 提示The package list for 'hackage.haskell.org' does not exist不存在，还需要执行

      cabal new-update
  因为速度慢切为代理执行
  
      proxychains cabal new-update
      
- 编译构建

      cabal new-build
      
- 测试运行

      cabal new-run
 
