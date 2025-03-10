# Takeaways from amazon COE review

* [Amazon's approach to failing successfully](https://www.youtube.com/watch?v=yQiRli2ZPxU) 
* [How AWS Minimizes the Blast Radius of Failures (ARC338)](https://www.youtube.com/watch?v=swQbA4zub20)
* [Amazon's approach to building resilient services](https://www.youtube.com/watch?v=KLxwhsJuZ44)
* [Why you should develop a correction of error (COE)](https://aws.amazon.com/cn/blogs/mt/why-you-should-develop-a-correction-of-error-coe/)
* [The incident is over: Now what?](https://www.youtube.com/watch?v=RaFx-4qrsJk&list=PL2yQDdvlhXf87XP-v6toFBLROmrJqApP7&index=7)

## What is COE?

Application reliability is critical. Service interruptions result in a negative customer experience, thereby reducing customer trust and business value. One best practice that we have learned at Amazon, is to have a standard mechanism for post-incident analysis. This lets us analyze a system after an incident in order to avoid reoccurrences in the future. These incidents also help us learn more about how our systems and processes work. That knowledge often leads to actions that help other incident scenarios, not just the prevention of a specific reoccurrence. The mechanism is called the `Correction of Error (COE)` process. Although post-event analysis is part of the COE process, it is different from a postmortem, because the focus is on corrective actions, not just documenting failures. This post will explain why you should start implementing the COE mechanism after an incident, and its components to help you get started.

## Some key points

1. 清晰的评估用户面影响（影响范围、影响特征等）
2. Good graph and clean metrics，产品的观测性是否足够
3. 防止问题再次发生 `5 why`来追寻根因而不是浮于现象
4. bug是不会消失的，必须有能力控制爆炸半径
   1. 从设计之初，就要注意region isolation 和 AZ independence
   2. region级别的元数据服务、控制面服务，也需要通过cell-based架构来隔离事故影响
   3. shuffle sharding来控制受损的用户比例
   4. 控制变更的范围和粒度（很多事故是由变更触发的）
5. 对于观测指标，需要明确`健康指标`和`诊断指标`。健康指标必须配置报警，用来指示服务是否存在异常；诊断指标则是用来判断为什么会发生异常。
6. 发生故障时，需要能够及时恢复：快速发现&快速止损
   1. 监控 & 告警 的有效性，经常review
   2. 自动化能力：基于告警的自动回滚、自愈、弹性&冗余设计