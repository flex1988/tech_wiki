# P 语言例子分析 1\_ClientServer

### 1 项目结构

```
.
├── ClientServer.pproj
├── portfolio-config.json
├── PSpec
│   └── BankBalanceCorrect.p
├── PSrc
│   ├── AbstractBankServer.p
│   ├── Client.p
│   ├── ClientServerModules.p
│   └── Server.p
└── PTst
    ├── TestDriver.p
    └── Testscript.p
```

1.  项目文件 ClientServer.pproj\


    ```
    <!-- P Project file for the Client Server example -->
    <Project>
    <ProjectName>ClientServer</ProjectName>
    <InputFiles>
    	<PFile>./PSrc/</PFile>
    	<PFile>./PSpec/</PFile>
    	<PFile>./PTst/</PFile>
    </InputFiles>
    <OutputDir>./PGenerated/</OutputDir>
    </Project>
    ```

    \
    P语言依赖于微软 .net运行时，pproj是一个描述工程的manifest文件，它是一个 xml格式的文件，声明了项目名称，和输入的源文件目录，以及输出的目录
2. portfolio-config.json 配置了一些默认参数，没有也能编，影响不大
3. PSpec目录定义了例子的两个规范
   1. **BankBalanceIsAlwaysCorrect（安全属性）保证了用户余额永远是对的**
   2. **GuaranteedWithDrawProgress（存活属性）保证了所有的取款请求都被响应了**\
