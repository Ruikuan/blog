# 在 Mac 系统下使用 Windows 习惯的快捷键

首先通过 `System Preferences - Keyboard - Modifier Keys` 修改修饰键为如下：

![modifier keys setting](https://raw.githubusercontent.com/Ruikuan/blog/master/Content/modifierKeysSetting.png)

这样就可以像在 Windows 下一样使用熟悉的 Ctrl A，Ctrl C 和 Ctrl V 了。

然后，对于文本编辑用户或者码农，Mac 的 Home 和 End 键是直接跳转到文档的开头和结尾，也是令人崩溃的。要想它像 Windows 下那样跳到段头和段尾，需要像下面这么设置。

创建或修改 `~/Library/KeyBindings/DefaultKeyBinding.dict` 文件，输入如下内容并保存：

```json
{
    "\UF729" = moveToBeginningOfLine:; // home
    "\UF72B" = moveToEndOfLine:; // end
    "$\UF729" = moveToBeginningOfLineAndModifySelection:; // shift-home
    "$\UF72B" = moveToEndOfLineAndModifySelection:; // shift-end
    "@\UF729" = moveToBeginningOfDocument:;
    "$@\UF729" = moveToBeginningOfDocumentAndModifySelection:;
    "@\UF72B" = moveToEndOfDocument:;
    "$@\UF72B" = moveToEndOfDocumentAndModifySelection:;
}
```
然后注销重新登录，就可以了。

### 附录
|Character code|Description|
|----------- | ----------- |
|\UF700	|Arrow up|
|\UF701	|Arrow down|
|\UF702	|Arrow left|
|\UF703	|Arrow right|
|\UF728	|Forward delete|
|\U007F	|Backward delete|
|\UF729	|Home|
|\UF72B	|End|
|\UF72C	|Page up|
|\UF72D	|Page down|
|\U001B |	Escape|
|\U0009	|Tab|
|\U0019	|Backtab (shift tab)|
|\U000D	|Return|
|\U0003	|Enter|

|Modifier Key Code|Description|
|----------- | ----------- |
|$|Shift (⇧)|
|@|Command (⌘)|
|^|Control (⌃)|
|~|Option (⌥)|
|#|Number pad|
 
默认的系统映射文件在这里：`System ▸ Library ▸ Frameworks ▸ AppKit.framework ▸ Resources ▸ StandardKeyBinding.dict`

> [Key bindings for switchers](https://macromates.com/blog/2005/key-bindings-for-switchers/)
> [NSStandardKeyBindingResponding](https://developer.apple.com/documentation/appkit/nsstandardkeybindingresponding)

```swift
func cancelOperation(Any?)
func capitalizeWord(Any?)
func centerSelectionInVisibleArea(Any?)
func changeCaseOfLetter(Any?)
func complete(Any?)
func deleteBackward(Any?)
func deleteBackwardByDecomposingPreviousCharacter(Any?)
func deleteForward(Any?)
func deleteToBeginningOfLine(Any?)
func deleteToBeginningOfParagraph(Any?)
func deleteToEndOfLine(Any?)
func deleteToEndOfParagraph(Any?)
func deleteToMark(Any?)
func deleteWordBackward(Any?)
func deleteWordForward(Any?)
func doCommand(by: Selector)
func indent(Any?)
func insertBacktab(Any?)
func insertContainerBreak(Any?)
func insertDoubleQuoteIgnoringSubstitution(Any?)
func insertLineBreak(Any?)
func insertNewline(Any?)
func insertNewlineIgnoringFieldEditor(Any?)
func insertParagraphSeparator(Any?)
func insertSingleQuoteIgnoringSubstitution(Any?)
func insertTab(Any?)
func insertTabIgnoringFieldEditor(Any?)
func insertText(Any)
func lowercaseWord(Any?)
func makeBaseWritingDirectionLeftToRight(Any?)
func makeBaseWritingDirectionNatural(Any?)
func makeBaseWritingDirectionRightToLeft(Any?)
func makeTextWritingDirectionLeftToRight(Any?)
func makeTextWritingDirectionNatural(Any?)
func makeTextWritingDirectionRightToLeft(Any?)
func moveBackward(Any?)
func moveBackwardAndModifySelection(Any?)
func moveDown(Any?)
func moveDownAndModifySelection(Any?)
func moveForward(Any?)
func moveForwardAndModifySelection(Any?)
func moveLeft(Any?)
func moveLeftAndModifySelection(Any?)
func moveParagraphBackwardAndModifySelection(Any?)
func moveParagraphForwardAndModifySelection(Any?)
func moveRight(Any?)
func moveRightAndModifySelection(Any?)
func moveToBeginningOfDocument(Any?)
func moveToBeginningOfDocumentAndModifySelection(Any?)
func moveToBeginningOfLine(Any?)
func moveToBeginningOfLineAndModifySelection(Any?)
func moveToBeginningOfParagraph(Any?)
func moveToBeginningOfParagraphAndModifySelection(Any?)
func moveToEndOfDocument(Any?)
func moveToEndOfDocumentAndModifySelection(Any?)
func moveToEndOfLine(Any?)
func moveToEndOfLineAndModifySelection(Any?)
func moveToEndOfParagraph(Any?)
func moveToEndOfParagraphAndModifySelection(Any?)
func moveToLeftEndOfLine(Any?)
func moveToLeftEndOfLineAndModifySelection(Any?)
func moveToRightEndOfLine(Any?)
func moveToRightEndOfLineAndModifySelection(Any?)
func moveUp(Any?)
func moveUpAndModifySelection(Any?)
func moveWordBackward(Any?)
func moveWordBackwardAndModifySelection(Any?)
func moveWordForward(Any?)
func moveWordForwardAndModifySelection(Any?)
func moveWordLeft(Any?)
func moveWordLeftAndModifySelection(Any?)
func moveWordRight(Any?)
func moveWordRightAndModifySelection(Any?)
func pageDown(Any?)
func pageDownAndModifySelection(Any?)
func pageUp(Any?)
func pageUpAndModifySelection(Any?)
func quickLookPreviewItems(Any?)
func scrollLineDown(Any?)
func scrollLineUp(Any?)
func scrollPageDown(Any?)
func scrollPageUp(Any?)
func scrollToBeginningOfDocument(Any?)
func scrollToEndOfDocument(Any?)
func selectAll(Any?)
func selectLine(Any?)
func selectParagraph(Any?)
func selectToMark(Any?)
func selectWord(Any?)
func setMark(Any?)
func swapWithMark(Any?)
func transpose(Any?)
func transposeWords(Any?)
func uppercaseWord(Any?)
func yank(Any?)
```
