## job 的作用
1. 创建一个或者多个 pod，并确保指定数量的 pod 可以成功地运行终止
2. 跟踪 pod 状态，根据配置及时重试失败的 pod
3. 确定依赖关系，保证上一个任务运行完毕后再运行下一个任务
4. 控制任务并行度，并根据配置确保 pod 队列大小

Job 控制器直接操作的就是 Pod


## job sepc
### restartPolicy
jobs.spec.template.spec.restartPolicy

### activeDeadlineSeconds
jobs.spec.template.spec.activeDeadlineSeconds

Job 的 pod 运行结束后，会进入 Completed 状态。可以通过参数 activeDeadlineSeconds 设置最长运行时间。一旦运行时间超过这个时间，Job 的所有 Pod 都会被终止

### parallelism
jobs.spec.parallelism

定义的是 Job 在任意时间最多可以启动多少个 pod 同时运行；

### completions
jobs.spec.completions

定义的是 Job 至少要完成的 pod 数目，即 Job 的最小完成数


## cron job 的作用
定时任务


## cron job spec

### concurrencyPolicy
cronjob.spec.concurrencyPolicy

由于定时任务的特殊性，很可能某个 job 还没有执行完，另外一个新 job 就产生了。可以通过这个参数来定义具体的处理策略

1. Allow，默认情况，表示这些 job 可以同时存在
2. Forbid，表示不会创建新的 pod，该创建周期被跳过
3. Replace，表示新产生的 job 会替换旧的、没有执行完的 job

### startingDeadlineSeconds
cronjob.spec.startingDeadlineSeconds

如果某一次 job 创建失败，这次创建就会被标记为 miss。当在指定的时间窗口内，miss 的数目达到 100 时，CronJob 就会停止再创建这个 Job。时间窗口可以由该参数指定

