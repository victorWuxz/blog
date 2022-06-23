**身为一个前端开发，在开发ts项目时，最繁琐的工作应该就是手写接口的数据类型和mock数据，因为这部分工作如果不做，后面写业务逻辑难受，做的话全是复制粘贴类似的重复工作，还挺费时间。下文将给大家介绍一个自动生成ts类型和mock数据的方法，帮助同学们从繁琐得工作中解脱出来。**

下面我们将通过一个示例，让大家一起了解一下代码生成的基本过程。
# TS代码生成基本流程

我们以下面这段ts代码为例，一起过一下生成它的基本流程。
```ts
export interface TestA {
  age: number;
  name: string;
  other?: boolean;
  friends: {
    sex?: 1 | 2;
  },
  cats: number[];
}
```

## 第一步：选定数据源
我们先思考一个问题，把上述代码写的interface生成需要哪些信息？

通过分析，我们首先需要知道它一共有几个属性，然后要知道哪些属性是必须的，除此以外还需要知道每个属性的类型、枚举等信息。有一种数据格式可以完美的给我们提供我们所需要的数据，它就是**JSON Schema**。

接触过后端的同学应该都了解过**JSON Schema**，它是对JSON数据的描述，举个例子，我们定义了下面这个JSON结构：
```json
{
  "age": 1,
  "name": "测试",
  "friends": {
  		"sex": 1
  },
  "cats": [
    1,
    2,
    3
  ],
  "other": true
}
```
我们口头描述下这个json：它有age、name、friends、cats、other5个属性，age属性的类型是number，name属性的类型是string，cats属性的类型是number组成的arry，friends属性是一个object，它有一个sex属性，类型是数字，other属性的类型是boolean。

用JSON Schema的描述如下：
```json
{
  "type": "object",
  "properties": {
    "age": {
      "type": "number"
    },
    "name": {
      "type": "string"
    },
    "cats": {
      "type": "array",
      "items": {
        "type": "number"
      }
    },
    "friends": {
      "type": "object",
      "properties": {
        "sex": {
          "type": "number"
        },
        "required": [
          "e"
        ]
      }
    },
		"other": {
			"type": "boolean",
		},
    "required": [
      "a",
      "b"
    ]
  }
}
```
可以看出**JSON Schema**可以完美的程序化实现我们的口头描述，这个例子比较简单，**JSON Schema**的描述能力远不止于此，比如**枚举，数组的最大长度，数字的最大最小值，是否是必须的**等我们常用的属性都能精确描述，所以它也常用于用户输入校验的场景。

## 第二步：选定代码生成工具
看到这个标题，相信大多数同学都已经知道了答案，没错，就是**TS AST**和[TS Compiler API](https://github.com/Microsoft/TypeScript/wiki/Using-the-Compiler-API)，后者可以生成或者修改**TS AST**，也可以输出编译后的文件。我们来看一下如何使用**TS Compiler API**生成抽象语法树并且编译成上文中提的代码。

对应的TS Compiler代码如下：
```ts
factory.createInterfaceDeclaration(
    undefined,
    [factory.createModifier(ts.SyntaxKind.ExportKeyword)],
    factory.createIdentifier("TestA"),
    undefined,
    undefined,
    [
      factory.createPropertySignature(
        undefined,
        factory.createIdentifier("age"),
        undefined,
        factory.createKeywordTypeNode(ts.SyntaxKind.NumberKeyword)
      ),
      factory.createPropertySignature(
        undefined,
        factory.createIdentifier("name"),
        undefined,
        factory.createKeywordTypeNode(ts.SyntaxKind.StringKeyword)
      ),
      factory.createPropertySignature(
        undefined,
        factory.createIdentifier("other"),
        factory.createToken(ts.SyntaxKind.QuestionToken),
        factory.createKeywordTypeNode(ts.SyntaxKind.BooleanKeyword)
      ),
      factory.createPropertySignature(
        undefined,
        factory.createIdentifier("friends"),
        undefined,
        factory.createTypeLiteralNode([factory.createPropertySignature(
          undefined,
          factory.createIdentifier("sex"),
          factory.createToken(ts.SyntaxKind.QuestionToken),
          factory.createUnionTypeNode([
            factory.createLiteralTypeNode(factory.createNumericLiteral("1")),
            factory.createLiteralTypeNode(factory.createNumericLiteral("2"))
          ])
        )])
      ),
      factory.createPropertySignature(
        undefined,
        factory.createIdentifier("cats"),
        undefined,
        factory.createArrayTypeNode(factory.createKeywordTypeNode(ts.SyntaxKind.NumberKeyword))
      )
    ]
 )
```
乍一看生成这段简单类型的代码非常复杂，但是仔细一看如果这些方法经过封装，代码会简洁不少，而且目前已经有一些比较成熟的第三方库库，比如[ts-morph](https://ts-morph.com)等。

Ts Compiler Api只有英文文档，而且使用复杂，而且生成不同类型的代码需要调用哪个函数我们不好确定，但我们可以去[TS AST View](https://ts-ast-viewer.com/#)查询，它能根据你输入的TS代码生成对应的抽象语法树和Compiler代码，上述代码就是TS AST View提供的。

`factory.createInterfaceDeclaration`方法会生成一个interface节点，生成之后，我们还需要调用一个方法将生成的interface打印出来，输出成字符串文件，参考代码如下：

```ts
// ast转代码
// 需要将上文factory.createInterfaceDeclaration生成的节点传入
export const genCode = (node: ts.Node, fileName: string) => {
    const printer = ts.createPrinter();
    const resultFile = ts.createSourceFile(fileName, '', ts.ScriptTarget.Latest, false, ts.ScriptKind.TS);
    const result = printer.printNode(
        ts.EmitHint.Unspecified,
        node,
        resultFile
    );
    return result;
};
```

## 第三步：美化输出的代码
美化代码这一步我们应该很熟悉了，相信我们编译器中都装有[Prettier](https://prettier.io/)，每个前端项目必备的工具，它不仅可以直接格式化我们正在编写的文件，**也可以格式化我们手动传入的字符串代码**，话不多说，上代码：
```ts
import * as prettier from 'prettier';

// 默认的prettier配置
const defaultPrettierOptions = {
    singleQuote: true,
    trailingComma: 'all',
    printWidth: 120,
    tabWidth: 2,
    proseWrap: 'always',
    endOfLine: 'lf',
    bracketSpacing: false,
    arrowFunctionParentheses: 'avoid',
    overrides: [
        {
            files: '.prettierrc',
            options: {
                parser: 'json',
            },
        },
        {
            files: 'document.ejs',
            options: {
                parser: 'html',
            },
        },
    ],
};

// 格式化美化文件
type prettierFileType = (content:string) => [string, boolean];
export const prettierFile: prettierFileType = (content:string) => {
    let result = content;
    let hasError = false;
    try {
        result = prettier.format(content, {
            parser: 'typescript',
            ...defaultPrettierOptions
        });
    }
    catch (error) {
        hasError = true;
    }
    return [result, hasError];
};
```

## 第四步：将生成的代码写入我们的文件
这一步比较简单，直接是有node提供的fs Api生成文件即可，代码如下：
```ts
// 创建目录
export const mkdir = (dir:string) => {
    if (!fs.existsSync(dir)) {
        mkdir(path.dirname(dir));
        fs.mkdirSync(dir);
    }
};
// 写文件
export const writeFile = (folderPath:string, fileName:string, content:string) => {
    const filePath = path.join(folderPath, fileName);
    mkdir(path.dirname(filePath));
    const [prettierContent, hasError] = prettierFile(content);
    fs.writeFileSync(filePath, prettierContent, {
        encoding: 'utf8',
    });
    return hasError;
};
```

# 前后端的协同
上面的流程还缺少重要的一步：**数据源JSON Schema谁提供？**

这就需要前后端的协同，目前后端已经有了很成熟的生成JSON Schema的工具，比如[Swagger](https://swagger.io/)，[YAPI](http://yapi.smart-xwork.cn/)等。

接入Swagger的后端系项目都能给前端提供swagger.json文件，文件的内容就包括所有接口的详细数据，包括JSON Schema数据。

YAPI和Swagger不同，它是API的集中管理平台，在它上面管理的api我们都可以通过它提供的接口获取的所有api的详细数据，和swagger.json提供的内容大同小异，而且YAPI平台支持导入或者生成swagger.json。

在这也偷偷吐槽一下我们目前的前后端协作流程中的问题：

1. 接口文档维护在wiki中，没有类似yapi这样的管理平台；
2. 接口数据格式没有一个统一的规范，或者规范落实不到位（老项目上这个问题较明显）；

如果有了接口管理平台和制定了相关规范，**前后端的协作效率会提升很多，减少沟通成本，而且前端也可以基于管理平台做一些工程效能相关的工作**。

# 难点攻克

上述步骤只是简单的介绍了一下生成ts类型代码的一个思路，这思路下还有有一些难点需要解决的，比如：

- 实际开发中我们需要注释，但TS Compiler API不能生成注释：**这个问题我们可以通过再代码的string生成之后然后在对应的地方手动插入注释的方式解决**。
- 实际业务的类型可能非常复杂，嵌套层次很深：**这个问题我们可以通过递归函数来解决**。
- 已经生成的类型代码，如果API有改动，应该怎么办，或者新增的API要和原来生成的放的一个文件下，这种情况怎么处理？**TS ComPiler API是可以读取源文件的，就是已经存在的文件也是可以读取的，我们可以读取源文件然后再利用Compiler API修改它的抽象语法树实现修改或者追加类型的功能**。
- 前后端的协同问题：这个就需要找leader解决了。

# 总结

经过上面提到的四个步骤，我们了解了生成代码的基本流程，而且每一步的实现方案不是固定的，可以自行选择：

- 在数据源选择的问题上，我们除了JSON Schema还可以选择原始的json数据当作数据源，只是生成的类型不是那么精准，在这推荐一个很好用的网站：[JSON2TS](http://json2ts.com/)。
- 代码生成工具我们也可以用常用的一些模板引擎来生成，比如[Nunjucks](https://nunjucks.bootcss.com/)，[EJS](https://ejs.bootcss.com/)等，它们不仅可以生成HTML，也可以**生成任何格式的文件**，并且能够返回生成的字符串。
- 代码美化这步还是推荐使用prettier。
- 对于前端来说，目前最好的输出文件的方式就是Node了。

> 本文只提供了一种工程化生成TS类型、Mock数据等简单可复制代码的思路，实现后能减少一部分**劳动密集型**的工作内容，让我们更专注于业务逻辑开发。

# 参考文献

- [TS Compiler API](https://github.com/Microsoft/TypeScript/wiki/Using-the-Compiler-API)
- [Prettier](https://prettier.io/)