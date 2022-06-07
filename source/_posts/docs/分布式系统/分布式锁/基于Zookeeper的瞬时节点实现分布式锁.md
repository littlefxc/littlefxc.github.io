---
title: 基于Zookeeper的瞬时节点实现分布式锁
date: 2021-05-12 21:32:54
tags:
- zookeeper
- 分布式锁
---

# Zookeeper 的数据结构

![](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/UE5cYm.png)

详细内容见官网

<!-- more -->

# 实现原理

![](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/JbLkYC.png)

![](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/o3SdiJ.png)

![](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/7D9zYJ.png)

![](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/X36dAV.png)

![](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/GUkIWX.png)

1. 线程A、B、C、D在zookeeper中的节点序号分别是1、2、3、4。
2. 节点序号最小的线程A 获得锁
3. 线程B 监听序号是1的节点（设置了一个观察器，监听节点1），线程C监听序号是2的节点，以此类推。
4. 线程A执行完任务后，序号为1的节点消失，线程B得到通知，线程B执行任务后，序号为2的节点消失，后续线程以此类推

# 代码实现

## ZkLock

```java
@Slf4j
public class ZkLock implements AutoCloseable, Watcher {

    private ZooKeeper zooKeeper;
    private String znode;

    public ZkLock() throws IOException {
        this.zooKeeper = new ZooKeeper("localhost:2181",
                10000,this);
    }

    public boolean getLock(String businessCode) {
        try {
            //创建业务 根节点
            Stat stat = zooKeeper.exists("/" + businessCode, false);
            if (stat==null){
                zooKeeper.create("/" + businessCode,businessCode.getBytes(),
                        ZooDefs.Ids.OPEN_ACL_UNSAFE,
                        CreateMode.PERSISTENT);
            }

            //创建瞬时有序节点  /order/order_00000001
            znode = zooKeeper.create("/" + businessCode + "/" + businessCode + "_", businessCode.getBytes(),
                    ZooDefs.Ids.OPEN_ACL_UNSAFE,
                    CreateMode.EPHEMERAL_SEQUENTIAL);

            //获取业务节点下 所有的子节点
            List<String> childrenNodes = zooKeeper.getChildren("/" + businessCode, false);
            //子节点排序
            Collections.sort(childrenNodes);
            //获取序号最小的（第一个）子节点
            String firstNode = childrenNodes.get(0);
            //如果创建的节点是第一个子节点，则获得锁
            if (znode.endsWith(firstNode)){
                return true;
            }
            //不是第一个子节点，则监听前一个节点
            String lastNode = firstNode;
            for (String node:childrenNodes){
                if (znode.endsWith(node)){
                    zooKeeper.exists("/"+businessCode+"/"+lastNode,true);
                    break;
                }else {
                    lastNode = node;
                }
            }
            synchronized (this){
                wait();
            }

            return true;

        } catch (Exception e) {
            e.printStackTrace();
        }
        return false;
    }

    @Override
    public void close() throws Exception {
        zooKeeper.delete(znode,-1);
        zooKeeper.close();
        log.info("我已经释放了锁！");
    }

    @Override
    public void process(WatchedEvent event) {
        if (event.getType() == Event.EventType.NodeDeleted){
            synchronized (this){
                notify();
            }
        }
    }
}
```

### 单元测试

```java
@RunWith(SpringRunner.class)
@SpringBootTest
@Slf4j
public class DistributeZkLockApplicationTests {

    @Test
    public void contextLoads() {
    }


    @Test
    public void testZkLock() throws Exception {
        ZkLock zkLock = new ZkLock();
        boolean lock = zkLock.getLock("order");
        log.info("获得锁的结果："+lock);

        zkLock.close();
    }
}
```

### 示例

```java
@RequestMapping("zkLock")
public String zookeeperLock(){
    log.info("我进入了方法！");
    try (ZkLock zkLock = new ZkLock()) {
        if (zkLock.getLock("order")){
            log.info("我获得了锁");
            Thread.sleep(10000);
        }
    } catch (IOException e) {
        e.printStackTrace();
    } catch (Exception e) {
        e.printStackTrace();
    }
    log.info("方法执行完成！");
    return "方法执行完成！";
}
```

程序启动两个，演示结果：

第一个程序的日志：

![](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/Y2fTpH.png)

第二个程序的日志：

![](https://raw.githubusercontent.com/littlefxc/littlefxc.github.io/images/images/rYNilY.png)

结论：第一个程序在11:04:51获得锁，10秒后即11:05:01释放锁，第二个程序在11:05:01获得锁，10妙后释放锁。ZkLock已经完成了我们的期望结果。

