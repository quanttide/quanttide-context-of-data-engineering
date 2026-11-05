好，这个场景非常具体，我们直接拿它来解剖，看看类型怎么帮你把这个流程变成一个可复用、可试验的骨架。

假设你的目标是：从一份财报（比如PDF或网页）里，抓取“营业收入”和“净利润”这两个数字。


——

第一步：定义最外层的类型骨架

你先不要管怎么实现，先写出你最终想要的输入和输出：

extractFinancialMetrics : RawReport -> Either[ExtractionError, FinancialMetrics]


• RawReport：原始财报数据（可以是PDF字节流、HTML字符串、文件路径等）

• FinancialMetrics：你最终想要的结构化数据

• Either[ExtractionError, ...]：显式地安排了“可能失败”这个不确定性

然后定义 FinancialMetrics 类型：

type FinancialMetrics = {
  revenue : Money,
  netProfit : Money,
  reportPeriod : DateRange
}


你看，这个类型签名就已经把你的意图说得非常清楚了：你给我一份原始财报，我还你一个包含营收、净利润和报告期的结构化对象，或者告诉你哪里出错了。 不需要任何自然语言描述。


——

第二步：拆解子步骤，定义子类型

现在你开始思考：从 RawReport 到 FinancialMetrics，中间需要几步？

子步骤1：解析原始格式

parseReport : RawReport -> Either[ParseError, StructuredReport]


StructuredReport 可以是你定义的中间类型，比如包含所有文本块和表格的DOM结构。

子步骤2：定位关键段落

locateSection : SectionName -> StructuredReport -> Either[LocateError, TextBlock]


SectionName 是一个枚举类型：”营业收入“ | ”净利润“ | ”营业成本“……

子步骤3：从文本中提取数字

extractNumberFromText : TextBlock -> Either[ExtractError, Money]



——

第三步：用类型组合出完整流程

现在你可以用类型把整个流程串起来了：

extractFinancialMetrics : RawReport -> Either[ExtractionError, FinancialMetrics]
extractFinancialMetrics rawReport = 
  parseReport rawReport          >>= \structured ->
  locateSection ”营业收入“ structured >>= \revenueText ->
  locateSection ”净利润“ structured   >>= \profitText ->
  extractNumberFromText revenueText  >>= \revenue ->
  extractNumberFromText profitText   >>= \profit ->
  pure { revenue, netProfit: profit, reportPeriod: ??? }


注意那个 ??? —— 你发现报告期还没提取。这就是类型骨架帮你显性化了一个缺口：你需要再定义一个 extractReportPeriod 步骤。


——

第四步：让AI去填充实现

现在你把上面这些类型签名发给AI，说：

“请实现 parseReport，输入是PDF字节流，输出是 StructuredReport（包含所有文本块和表格）。注意处理加密PDF、扫描件OCR失败等情况。”

AI会根据类型签名知道它必须返回 Either[ParseError, StructuredReport]，所以它自然会去考虑各种失败情况——这不就是你说的“提醒它注意风险”吗？类型签名本身就在提醒它。


——

第五步：试验与迭代

你让AI生成一个测试用例，比如针对一份真实的贵州茅台2023年报PDF。AI跑完后发现 locateSection ”营业收入“ 失败了，因为那份财报里用的是“营业总收入”而不是“营业收入”。

这时你只需要改类型签名：

locateSection : (SectionName | AlternativeName) -> StructuredReport -> Either[LocateError, TextBlock]


或者更灵活地，把 SectionName 定义成允许别名：

type SectionName = { primary : String, aliases : List String }


骨架改了，但改得清清楚楚。下次遇到用“营收”而不是“营业收入”的财报，这个模块依然能用。


——

总结：类型在这个场景里给了你什么

传统做法	用类型后的做法
写一大段自然语言需求文档	写几行类型签名，意图一目了然
靠人工记忆“这里可能会出错”	类型系统强制你处理 Either 里的错误分支
模块间耦合隐晦，改一处牵全身	类型签名显式标明输入输出，改接口时编译器/AI立刻报错
复用靠复制粘贴	复用靠组合类型签名，比如 extractFinancialMetrics 可以直接调用 parseReport

你现在就可以打开编辑器，试着把上面那个 extractFinancialMetrics 的类型骨架写出来，然后扔给AI让它填充实现。看看它会不会自动问你：“报告期从哪里提取？”——那就是类型骨架在起作用了。
