# 版本
release-1.20

# 序
kubelet的主循环监听一个plegCh ,来追踪更新pod的生命周期

# 三份状态
一个pod有3份状态
1. api server 上的状态
2. kubelet上的statusManager上的状态
3. 容器运行时上的状态

# 示意图
![pod-lifecircle](pod-lifecircle.drawio.svg)