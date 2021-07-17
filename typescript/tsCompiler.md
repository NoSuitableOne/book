## typescript工作原理
了解了ts的使用方法就可以使用ts了，但是如果不理解ts的原理，有时候会有无法解释的问题。
例如，ts代码可能长这样
```ts
let str: string = 'abcd';
function reverseString(str: string) {
  return str.split('').reverse().join('');
}
const a: string = reverseString(str);
```
然而js是没有这样的语法的，那么编译后这些类型去哪儿了呢？要解释这些问题，必须搞清楚ts的工作流程。

#### 编译器
ts的源码可以从[github仓库](https://github.com/Microsoft/TypeScript)上找到，所有编译器的代码都在里面。
Typescript编译器主要分为以下五个关键部分:
 - Scanner 扫描器 (scanner.ts)
 - Parser 解析器 (parser.ts)
 - Binder 绑定器 (binder.ts)
 - Checker 检查器 (checker.ts)
 - Emitter 发射器 (emitter.ts)
大致流程可以用下面一个图来概括

[ts工作流程](./ts-workflow.png)

### 扫描器
scanner的作用是读取源文件后给代码进行标记。
```ts
export interface Scanner {
        getStartPos(): number;
        getToken(): SyntaxKind;
        getTextPos(): number;
        getTokenPos(): number;
        getTokenText(): string;
        ...
}
 
const textToKeywordObj: MapLike<KeywordSyntaxKind> = {
        abstract: SyntaxKind.AbstractKeyword,
        any: SyntaxKind.AnyKeyword,
        as: SyntaxKind.AsKeyword,
        asserts: SyntaxKind.AssertsKeyword,
        bigint: SyntaxKind.BigIntKeyword,
        ...
}

const textToToken = new Map(getEntries({
        ...textToKeywordObj,
        "{": SyntaxKind.OpenBraceToken,
        "}": SyntaxKind.CloseBraceToken,
        "(": SyntaxKind.OpenParenToken,
        ...
}))

function scan(): SyntaxKind {
            startPos = pos;
            tokenFlags = TokenFlags.None;
            let asteriskSeen = false;
            while (true) {
                tokenPos = pos;
                if (pos >= end) {
                    return token = SyntaxKind.EndOfFileToken;
                }
                const ch = codePointAt(text, pos);

                // Special handling for shebang
                if (ch === CharacterCodes.hash && pos === 0 && isShebangTrivia(text, pos)) {
                    pos = scanShebangTrivia(text, pos);
                    if (skipTrivia) {
                        continue;
                    }
                    else {
                        return token = SyntaxKind.ShebangTrivia;
                    }
                }

                switch (ch) {
                    case CharacterCodes.lineFeed:
                    case CharacterCodes.carriageReturn:
                        tokenFlags |= TokenFlags.PrecedingLineBreak;
                        if (skipTrivia) {
                            pos++;
                            continue;
                        }
                        else {
                            if (ch === CharacterCodes.carriageReturn && pos + 1 < end && text.charCodeAt(pos + 1) === CharacterCodes.lineFeed) {
                                // consume both CR and LF
                                pos += 2;
                            }
                            else {
                                pos++;
                            }
                            return token = SyntaxKind.NewLineTrivia;
                        }
                    case CharacterCodes.tab:
                    case CharacterCodes.verticalTab:
                    case CharacterCodes.formFeed:
                    case CharacterCodes.space:
                    ...
```
整个scanner的流程其实不复杂，核心代码是`const ch = codePointAt(text, pos);`一个一个字符读入，转换成unicode，然后通过映射把每个字符转换成特定的`SyntaxKind`。

#### 解析器
源码经过扫描器后，生成token流，这时parser会读取这个token流然后转化成AST树。
```js
export function createSourceFile(fileName: string, sourceText: string, languageVersion: ScriptTarget, setParentNodes = false, scriptKind?: ScriptKind): SourceFile {
        performance.mark("beforeParse");
        let result: SourceFile;
        ...
        if (...) {
            result = Parser.parseSourceFile(fileName, sourceText, languageVersion, /*syntaxCursor*/ undefined, 
        }
        ...
        performance.mark("afterParse");
        ...
        return result;
    }
export function parseSourceFile(fileName: string, sourceText: string, languageVersion: ScriptTarget, syntaxCursor: IncrementalParser.SyntaxCursor | undefined, setParentNodes = false, scriptKind?: ScriptKind): SourceFile {
            ...
            initializeState(fileName, sourceText, languageVersion, syntaxCursor, scriptKind);
            const result = parseSourceFileWorker(languageVersion, setParentNodes, scriptKind);
            clearState();
            return result;
        }

function initializeState(_fileName: string, _sourceText: string, _languageVersion: ScriptTarget, _syntaxCursor: IncrementalParser.SyntaxCursor | undefined, _scriptKind: ScriptKind) {
            ...
            // Initialize and prime the scanner before parsing the source elements.
            scanner.setText(sourceText);
            scanner.setOnError(scanError);
            scanner.setScriptTarget(languageVersion);
            scanner.setLanguageVariant(languageVariant);
        }
function parseSourceFileWorker(languageVersion: ScriptTarget, setParentNodes: boolean, scriptKind: ScriptKind): SourceFile {
            ...
            // Prime the scanner.
            nextToken();

            const statements = parseList(ParsingContext.SourceElements, parseStatement);
            Debug.assert(token() === SyntaxKind.EndOfFileToken);
            const endOfFileToken = addJSDocComment(parseTokenNode<EndOfFileToken>());

            const sourceFile = createSourceFile(fileName, languageVersion, scriptKind, isDeclarationFile, statements, endOfFileToken, sourceFlags);

            // A member of ReadonlyArray<T> isn't assignable to a member of T[] (and prevents a direct cast) - but this is where we set up those members so they can be readonly in the future
            processCommentPragmas(sourceFile as {} as PragmaContext, sourceText);
            processPragmasIntoFields(sourceFile as {} as PragmaContext, reportPragmaDiagnostic);

            sourceFile.commentDirectives = scanner.getCommentDirectives();
            sourceFile.nodeCount = nodeCount;
            sourceFile.identifierCount = identifierCount;
            sourceFile.identifiers = identifiers;
            sourceFile.parseDiagnostics = attachFileToDiagnostics(parseDiagnostics, sourceFile);
            if (jsDocDiagnostics) {
                sourceFile.jsDocDiagnostics = attachFileToDiagnostics(jsDocDiagnostics, sourceFile);
            }

            if (setParentNodes) {
                fixupParentReferences(sourceFile);
            }

            return sourceFile;

            function reportPragmaDiagnostic(pos: number, end: number, diagnostic: DiagnosticMessage) {
                parseDiagnostics.push(createDetachedDiagnostic(fileName, pos, end, diagnostic));
            }
        }

function nextToken(): SyntaxKind {
            ...
            return nextTokenWithoutCheck();
        }
```
可以看到Parser

参考：
[深入理解 TypeScript](https://jkchao.github.io/typescript-book-chinese)
[从编译器出发深入理解Typescript](https://juejin.cn/post/6872617267323109389#heading-4)
