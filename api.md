## 深入API 

系统设计的重点

### Principles

 - All API 应该是 declaritive
 - API objects 应该是互补的或者composable，不应该是模糊的包装
 - 控制面应该是透明的，不应该有隐藏的内部API
 - API 操作的cost应该和要操作的资源数量成正比。所以，一般的查询必须要有索引。当心某些模式下多个API调用或许会导致指数级cost.
 - 对象状态必须为100%重构by observation。如果保存有历史数据，则其必须只是用于系统优化。correction operation不应该依赖于历史数据。
 - 保证cluster-wide 不变量正确并不容易，因此应该尽量避免使用。如果无法避免，不要enforce them atomically in master components. which is contention prone，并且如果存在bug修改了invariant，那么甚至没有一个恢复的方法。应该提供一系列的检测来降低vialation的概率，并且确保所有相关的component都有从invariant vialation中恢复的方法。
 - 底层的API应该设计为由上层的系统控制。上层API应该