---
title: "每日基础技术总结 · 2026-06-05 · SAML 断言签名绕过与 XML 签名包装攻击"
date: 2026-06-05 08:00:00
categories: [技术分享]
tags: ["技术分享", "安全基础"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2026-06-05 · SAML 断言签名绕过与 XML 签名包装攻击

## 📚 今日主题

> **SAML 断言签名绕过与 XML 签名包装攻击**（安全基础）

### 1. 核心概念速览
SAML 断言签名绕过与 XML 签名包装攻击本质上是签名验证范围与数据消费范围不一致导致的信任边界漏洞。XML 签名（XMLDSig）只对通过 URI 引用（Reference）定位的特定节点子集计算摘要并签名，并不保证整个 XML 文档的完整性和真实性。攻击者利用该机制，将合法签名的断言节点复制或移动到新构造的 XML 文档中，同时在文档其他位置注入恶意但未签名的断言，使依赖方（SP）在验证时找到合法签名节点并通过校验，但在后续解析时却使用攻击者注入的未签名节点。该攻击位于安全基础中的『完整性验证与解析分离』问题域，与 JWT 等整体签名机制形成鲜明对比。专业工程师必须掌握它，因为任何基于 XML 的联邦协议（SAML、WS-Security）都可能存在此漏洞，且现代云身份体系仍大量使用 SAML 2.0，理解其底层缺陷是构建安全身份基础设施的前提。

### 2. 底层原理剖析
XML 签名规范（XMLDSig）的核心流程：签名者选取一个或多个资源（Resource），用 URI 片段标识符（如 #assertion-id）指向 XML 文档中的元素；对每个资源执行规范化（Canonicalization）和摘要（Digest），生成 Reference；再对包含所有 Reference 的 SignedInfo 节点做签名。验证方解析签名时，根据 SignedInfo 中的 URI 在当前 XML 文档中查找对应 ID 的元素，重新计算摘要并与 DigestValue 比较；若一致，则签名验证通过。

攻击的关键：验证方在验证完成后，解析整个 XML 文档以提取 SAML 断言，但 XML 解析器不会自动区分『哪个断言被签名过』。攻击者构造如下文档：

1. 将原始合法断言（含原签名）整体放入文档的一个位置；
2. 在文档的另一个位置放入攻击者构造的恶意断言（修改了权限属性或主题）；
3. 将 Signature 节点中的 URI 指向合法断言，使验证时找到合法节点并验证通过；
4. 应用代码随后使用 XPath 或 DOM 查询断言时，可能返回第一个匹配的断言（即恶意断言），或返回节点集合中的最后一个，取决于实现。

本质上是『引用验证』与『实际使用』两个动作作用于不同对象。前端类比：Java 的接口与 TypeScript 的接口虽然都叫接口，但 Java 接口是运行时类型约束和契约，TS 接口是编译时结构约束，两者在编译后消失；同样，SAML 断言签名只是『指针级』验证，而应用逻辑解析的是整个文档，二者是不同层面的事实。理解这一点，就能意识到验证和解析必须基于同一个选取逻辑，否则存在语义鸿沟。

### 3. 基础代码与实战验证
```text
# 极简 Python 伪代码演示签名验证与数据提取的分离
import xml.etree.ElementTree as ET

def verify_signature(xml_doc, signature_node):
    # 1. 从 Signature 节点中取出 Reference 的 URI，例如 "#assertion1"
    uri = signature_node.findtext('.//Reference')
    target_id = uri[1:]  # 去掉 '#'

    # 2. 在整个文档中查找具有对应 ID 的元素（合法断言）
    target = xml_doc.find(f'.//*[@ID="{target_id}"]')

    # 3. 对 target 做规范化并计算摘要，与 DigestValue 比较
    digest = compute_digest(canonicalize(target))
    if digest != signature_node.findtext('.//DigestValue'):
        return False
    return True

def extract_assertion(xml_doc):
    # 应用代码通常直接查找所有断言节点，取第一个或按某种规则
    # 攻击者注入的恶意断言可能出现在合法断言之前
    return xml_doc.findall('.//Assertion')[0]

# 攻击者构造的文档：包含两个 Assertion，第一个是恶意未签名，第二个是合法已签名
malicious = ET.fromstring("""
<Response>
  <Assertion ID="evil">
    <Attribute Name="role">admin</Attribute>  <!-- 未签名内容 -->
  </Assertion>
  <Assertion ID="legit">
    <Attribute Name="role">user</Attribute>   <!-- 原始合法内容 -->
  </Assertion>
  <Signature>
    <Reference URI="#legit"/>  <!-- 验证时指向合法节点 -->
  </Signature>
</Response>""")

# 验证通过，但 extract_assertion 返回第一个 Assertion，即 evil
if verify_signature(malicious, malicious.find('.//Signature')):
    assertion = extract_assertion(malicious)
    print(assertion.attrib['ID'])  # 输出 evil，完成绕过

# 注意：真实攻击利用 XML 解析器的 ID 属性类型、规范化差异等，但核心原理一致
```

### 4. 常见误区与进阶思考
常见误区一：认为 XML 签名能保证整个文档的完整性。实际上 XMLDSig 默认只保护被 URI 引用的节点，除非使用 SignedInfo 中的 Transformation 显式包含整个文档。工程师必须验证签名后，再对实际使用的节点进行完整性绑定，例如要求提取的断言 ID 必须等于签名 Reference 指向的 ID，并且该节点必须是文档的唯一顶层业务元素。

常见误区二：依赖 XML 解析器的默认 ID 查找行为。不同解析器对 ID 属性的识别可能不同（有的只认 'ID'，有的也认 'xml:id'），攻击者可利用这些差异构造签名验证时定位到合法节点，而数据提取时定位到恶意节点。正确做法是显式指定签名 Reference 的 URI 必须指向业务根节点，并在代码中强制提取的节点与签名的节点是同一个对象（如通过节点引用而非重新查询）。

深度思考题：如果 SAML 响应中既包含签名节点，又包含一个经过 XML 规范化处理后的相同断言（仅空白字符不同），攻击者能否让签名验证通过且让解析器提取到未签名的副本？请分析 XML 规范化（Canonicalization）在签名验证和解析器默认处理方式上的差异，并设计一种防御机制，确保签名验证与业务数据提取的节点身份一致，且能抵抗 ID 属性混淆和节点复制攻击。
