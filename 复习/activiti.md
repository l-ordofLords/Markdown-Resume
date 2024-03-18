1.activiti统计一个人有多少个流程正在他身上审批

2.activiti如何实现一个会签

```xml
<userTask id="usertask2" name="领导审批" activiti:assignee="${leader}">
      <extensionElements>
        <activiti:taskListener event="complete" class="com.demo.activiti.SignTaskListener"></activiti:taskListener>
      </extensionElements>
      <multiInstanceLoopCharacteristics isSequential="false" activiti:collection="${leaderList}" activiti:elementVariable="leader">
        <completionCondition>${nrOfCompletedInstances/nrOfInstances == 1}</completionCondition>
      </multiInstanceLoopCharacteristics>
    </userTask>
```

【说明】：

1、isSequential=”false” 表示这是非串行会签，即为并行会签，如三个人参与会签，是三个人同时收到待办，任务实例是同时产生的。

2、activiti:collection 表示是会签的参与人员集合，用户可以通过定义自身的服务类来获取

3、completionCondition 表示是任务往下跳转的完成条件，返回true是，表示条件成立，流程会跳至下一审批环节。

activiti多实例它一开始就初始化好跟activiti:collection个数相同的task，后续无法做到任意加人会签