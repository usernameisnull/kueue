**kueue源码摘要**

- 一些配置:

    1.1 Development: true, 输出更好阅读  
    1.2 '--zap-log-level=6' 输出更多日志

- top:rf = 1:n, 当删除top时,需要查看是否有关联的rf,如果有就不能删除,所以同时需要监听rf的删除事件, 当top, rf发生变化时,会影响到与之关联的cq上的wokload,可能会从inadmissable状态转移到heap中,以便在scheduler中处理.
- 当创建一个新top时,如果有rf绑定了它,则会在flavorCache中,创建一个以rf name 为key,的 TASFlavorCache对象,此对象有相应的rf, top的部分spec信息,用于在后边的tas调度.
- 当创建lq时,会查看与之关联的cq是否存在,如果不存在则不可启用,如果存在且有workloads,且wl的状态也正常,则将之放置到相应的cq的heap上,以使在scheduler中被pop后处理.
- 当创建cq时,需要根据不同的特征,启用不同处理,比如snapshot(cq:sp = 1:1 )功能,需要在visiblity启用时才起使用,他是将此队列当前拥有的inflight(已经从heap中pop出来) + heap中的 + inadmissable queue中的所有wl的总和,同时在此会对当前已经使用的资源usage之类的进行统计,并处理resourceGroup等.并且会按lq来分组统计usage,一个lq的usage等于,提交到它的所有wl的usage的总和
- 当创建pytorchJob时,会创建对应的Workload,或者使用已有的workload来更新以匹配,并配置其调度优先级什么的, 并**添加调度门**,(_后文要用)_ 当他删除时,会从对应workload上移除finalizer
- 当创建workload时,逻辑在此处比较简单,所有的操作都会转向scheduler.go中的相应方法
- **scheduler.go (关键)**

    8.0 从heap中将每个cq的第一个wl pop出来,作为本次调度要处理的wl
    
    8.1 如果启用了tas, 基于top生成一个top树，并统计已经使用的usage与cap,在统计容量时,必须减去非tas使用的资源,pod本身也是资源 ,供7.2筛选使用: snapshotForNodes
    
    8.2 为当前的所有cq生成快照,如果某个cq使用了8.1中的某些rf,是将其添加到cq的快照中: snapshotClusterQueue
    
    8.3 为当前cq的所有rf生成快照,(_后面所有的运算都是基于此处的快照数据)_
    
    8.4 然后进入nominate阶段

        8.4.1 先进行resource, limit等各种检测
    
        8.4.2 构建flavorassigner用以assign, 在assign时,为此workload,计算/更新cq/lq中保存的usage, cap相关信息,算出tas相关需求
    
        8.4.3 为podSet 执行实际的assign: findTopologyAssignment
    
            8.4.3.1 先做各种check,比如taint, toleration相关的
    
            8.4.3.2 先统计出每个domain能容纳的pod的个数,构成一棵树: fillInCounts
    
            8.4.3.3 算出合适的level级别,注意排序优先级, wl被分为leader与worker两组(分成两组的原因,应该是满足lws),有前者的优先级更高,同时支持xx**StateSlice**与xx**State**两级调度**(State即是pod的个数)**,且Slice级优先级更高,即先assgin leader,再是worker: findLevelWithFitDomains
    
            8.4.3.4 基于选择的算法优化前一步的结果
    
            8.4.3.5 构建最终的assigment: buildAssignment
    
        8.4.4 将wl分成可 (in)admissable 两组
    
    8.5 进入admit阶段
    
        8.5.1 为workload reserve配额
    
        8.5.2 为workload assume 资源(就是已经确定但还未经过admission确认的): AssumeWorkload
    
        8.5.3 向服务器 apply前一步的assume: applyAdmission
    
        8.5.4 进入admited
    
    8.6 goingOn
    
    - topologyUngater 中会监听第8步中对workload的处理,并将admited的pod上的nodeSelector改成第8步的结果,并**删除调度门**,到此pod成功调度,一次调度完成