---
title: "每日基础技术总结 · 2024-04-27 · pytest：fixture、参数化与 mock"
date: 2024-04-27 20:00:00
categories: [技术分享]
tags: ["技术分享", "Python 工程化"]
author: Litongjin
disableNunjucks: true
---

# 每日基础技术总结 · 2024-04-27 · pytest：fixture、参数化与 mock

## 📚 今日主题

> **pytest：fixture、参数化与 mock**（Python 工程化）

### 1. 核心概念速览
pytest 的 fixture 是依赖注入容器的实现机制，通过作用域（scope）生命周期管理资源的生命周期，解决测试间的状态隔离与共享问题；参数化（parameterization）通过运行时动态生成测试用例实例，将数据驱动测试（DDT）逻辑显式化，避免代码冗余；mock 通过替换对象属性或方法引用指向模拟对象，切断真实 I/O 或外部服务调用，实现单元测试的纯净性与确定性。这三者共同构成 Python 工程化中不可变测试环境的基石，确保测试的可重复性、隔离性和执行效率，是后端服务稳定性保障的核心工具链。

### 2. 底层原理剖析
1. Fixture 机制：本质是带标记的工厂函数。pytest 解析器扫描测试函数签名，匹配同名或间接引用的 fixture 名。根据 scope（function/class/module/session/pkg）决定何时创建实例及何时销毁。若多个测试依赖同一 fixture 且 scope 较大，则在首次请求时初始化，后续复用直至生命周期结束，利用装饰器 @pytest.fixture 注册到全局命名空间。
2. 参数化机制：@pytest.mark.parametrize 在字节码编译期或导入期展开，为每个输入组合生成独立的测试类实例。底层通过继承 TestCase 或使用钩子机制注入不同参数，确保每个用例拥有独立的 pytest 运行上下文。
3. Mock 机制：基于 sys.modules 和对象属性的动态替换。unittest.mock.patch 是一个上下文管理器，它在进入作用域时将目标路径的对象指针替换为 MagicMock/Mock 实例，保存原始引用，退出后恢复。这直接操作了内存中的引用关系，而非修改代码逻辑。与前端 TS/JS 的 mock server (Nock) 不同，pytest mock 是在进程内（In-Process）进行引用劫持，延迟更低，无需网络开销。

### 3. 基础代码与实战验证
```text
import pytest
from unittest.mock import patch, MagicMock

def test_fixture_scope_and_mock():
    '''验证 fixture 的生命周期管理以及 mock 对内部方法的拦截'''
    
    class Service:
        def get_data(self): return "original"

    # Fixture: scope='module' 意味着整个模块仅初始化一次
    @pytest.fixture(scope='module')
    def service_instance():
        s = Service()
        yield s
        print("Teardown: resource cleanup")

    # Parameterize: 传入两组不同的输入数据
    @pytest.mark.parametrize("input_val", [10, 20])
    def test_logic_with_mock(input_val, service_instance):
        # Mock Patch: 将 Service.get_data 替换为返回固定值的 Mock 对象
        with patch.object(service_instance, 'get_data', return_value="mocked_res"): 
            # 此时调用 get_data 不再执行原逻辑，而是执行 MagicMock 的逻辑
            result = service_instance.get_data()
            assert result == "mocked_res"
            # 内部逻辑验证可在此处断言 input_val 如何影响其他非 IO 行为

# 伪代码描述底层流程：
# 1. pytest 启动 -> 发现 test_logic_with_mock
# 2. 解析 params [(10,), (20,)] -> 生成两个测试实例 TestClass_10, TestClass_20
# 3. 检查 fixture 'service_instance' scope='module' -> 未初始化 -> 创建实例 S1 -> S1 入缓存
# 4. 执行 TestClass_10 -> 进入 patch 上下文 -> S1.get_data 引用被替换 -> 断言成功 -> 退出 patch -> 引用恢复
# 5. 执行 TestClass_20 -> 发现 S1 已存在 -> 直接使用 S1 -> 进入 patch -> 断言成功 -> 退出 patch
# 6. 模块结束 -> 清理 module scope 资源 -> 打印 Teardown
```

### 4. 常见误区与进阶思考
误区一：混淆 'Mocking 返回值' 与 'Mocking 副作用'。很多工程师只关注 return_value，忽略了 mock 对象会捕获所有调用次数（call_count）和调用参数（call_args），若未正确断言这些元数据，可能导致测试通过了但实际交互错误的盲区。误区二：Fixture 的作用域滥用导致状态污染。若在高开销的 session 级 fixture 中创建了可变对象（如列表或数据库连接），且在测试中对该对象进行了原地修改（in-place mutation），后续测试将读取到被篡改的状态，导致间歇性失败。思考题：如果在一个 session-scoped 的 fixture 中打开了一个数据库事务并等待提交，而后续的 function-scoped 测试需要回滚该事务，pytest 的原生 fixture 机制如何实现这种细粒度的资源生命周期嵌套控制？（提示：涉及 yield 的位置与 teardown 的触发时机）
