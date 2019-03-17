### WAL（Write Ahead Logging） 预写日志
#### 实现目标：
+ 1.并发txns的多线程日志记录和恢复
+ 2.单线程检查点创建和恢复
+ 3.在事务执行期间进行轻量级日志记录，仅记录提交txns
+ 4.协作日志恢复和检查点恢复
+ 5.按检查点截断/最小化日志文件

#### 日志模型设计  
&emsp;&emsp;&emsp;&emsp;&emsp;![](/images/wal1.png)   
&emsp; 日志管理器控制所有后端记录器和前端记录器。它提供了查询和管理这些记录器的接口。在 Peloton 启动期间, 日志管理器读取日志设置的配置文件。例如应该使用哪种日志协议。  
&emsp;后端记录器是线程本地实例。 它负责收集工作线程中生成的所有日志。  
&emsp;前端记录器是一个全局单例实例。记录管理器确保所有新创建的后端记录器都链接到前端记录器。然后，前端记录器不断从所有已注册的后端记录器收集日志，并将日志记录刷新到日志文件或其他持久性存储。   
#### 日志模块工作流程(workflow)和重要接口  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;![](/images/wal2.png)   
#### 恢复协议  
+ 1.恢复检查点
+ 2.找到最小的PCID日志
+ 3.在检查点ID和持久提交ID之间恢复日志中的事务
+ 4.重建索引

#### 日志状态图  
+ 0.无效(invalid)
+ 1.Standby -- Bootstrap（待机--引导）
+ 2.恢复(Recovery) - 可选恢复
+ 3.记录(Logging) - 收集数据并在提交时刷新
+ 4.终止(Terminate) - 收集任何剩余数据并刷新
+ 5.睡眠(Sleep) - 从管理器断开后端记录器和前端记录器

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;![](/images/wal3.png)
### WBL(Write Behind Logging) 后写日志
#### 日志文件中没有元组数据
&emsp;日志中只记录元组标头信息。 我们使用数据库ID、表ID和元组偏移量来引用元组数据存储中的元组数据。
#### 后写日志  
&emsp;所有修改都将直接应用于 NVRAM, 而无需等待日志被刷新。
&emsp;我们依靠MVCC来确保一致性。因此，虽然更新的数据会立即写入持久存储，但在刷新所有日志项之前它们是不可见的。当所有日志都保持不变时，DBMS会设置数据库的版本以使所有挂起的更新可见。我们通过使用提交位（插入位和删除位）来控制元组更新可见性。只有在刷新日志之后，日志记录线程才会开始更改元组中的提交位。  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;![](/images/wal4.png)   
#### 执行WBL时的5个步骤事务工作流:
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;![](/images/wal5.png)    
#### 扩展日志工作流程   
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;![](/images/wal6.png)  
#### 典型的Peloton日志记录文件结构：
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;![](/images/wal7.png)    
#### 恢复  
&emsp;确保在更改COMMIT位期间不中断上一次运行。 否则，重做更改提交位。
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;![](/images/wal8.png)  
#### 单元测试   
&emsp;测试命令:
+ cd build/tests
+ ./logging_test
+ ./checkpoint_test    

#### 测试用例  
&emsp; writing_logfile：执行事务并写入日志。
&emsp; 恢复:关闭日志记录并写入更多虚拟记录。 然后重置数据库并恢复。
