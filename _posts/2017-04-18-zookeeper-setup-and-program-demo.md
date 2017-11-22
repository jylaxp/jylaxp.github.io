---
layout: post
title: zookeeper 配置和编程示例
---

## 1. zookeeper配置
### 1.1 配置文件
zk的配置文件样例是conf/zoo_sample.cfg， 在zk启动的时候默认读取的配置文件是zoo.cfg。   
copy一份zoo_sample.cfg为zoo.cfg
```shell
cp zoo_sample.cfg zoo.cfg
```
默认的配置文件内容
```shell
# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial 
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between 
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just 
# example sakes.
dataDir=/tmp/zookeeper
# the port at which the clients will connect
clientPort=2181
# the maximum number of client connections.
# increase this if you need to handle more clients
#maxClientCnxns=60
#
# Be sure to read the maintenance section of the 
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1
```
在zk的数据目录创建myid文件，这里是 /tmp/zookeeper 
```shell
echo 1 >> /tmp/zookeeper/myid
```
myid文件是zk节点在集群中的编码，范围是:1~255   
### 1.2 配置集群
zk的集群节点信息是配置在zoo.cfg中的，节点信息格式是
```shell
server.x=ip:port:port
```
- server : 固定写法
- x：这个x表示节点id，就是上节中配置在myid文件中的数字
- ip: 节点的ip地址
- port: 第一port，集群用来同步数据，处理事务通信；第二个port用于集群选举   
如果有三个节点，它们的节点信息分别为
- node1:  192.168.10.11:2888:3000, id为1
- node2:  192.168.10.12:2888:3000, id为2
- node3:  192.168.10.13:2888:3000, id为3
让它们三个组成一个集群，则需要在这三个节点的配置文件中加入如下配置项
```shell
server.1=192.168.10.11:2888:3000
server.2=192.168.10.12:2888:3000
server.3=192.168.10.13:2888:3000
```
启动三个节点，自动组成集群。
### 1.3 单机模式
单机模式就很简单了，只有一个节点的集群就是单机模式。

# 2. zookeeper API 使用
## 2.1 同步方法调用示例   
测试接口
```java
package com.mydomain;

/**
 * @author jyl25609
 * @version Id: Test, v 0.1 2017/4/18 10:03 jyl25609 Exp $
 */
public interface Test {
    /**
     * 测试方法
     */
    void test();
}
```
zookeeper API 测试抽象类
```java
package com.mydomain.zookeeper;

import java.util.concurrent.CountDownLatch;

import org.apache.log4j.Logger;
import org.apache.zookeeper.WatchedEvent;
import org.apache.zookeeper.Watcher;
import org.apache.zookeeper.ZooKeeper;
import org.apache.zookeeper.data.Stat;

import com.mydomain.Test;

/**
 * @author jyl25609
 * @version Id: ZooTest, v 0.1 2017/4/17 15:30 jyl25609 Exp $
 */
public abstract class ZooTest implements Watcher, Test {

    /** 日志 */
    protected Logger              logger         = Logger.getLogger(ZooTest.class);

    /** countDownLatch */
    private static CountDownLatch countDownLatch = new CountDownLatch(1);

    /** countDown */
    private static CountDownLatch countDown      = new CountDownLatch(1);

    /** 服务器 */
    private final static String   server         = "192.168.95.128:2181";

    /** 超时时间 */
    private final static int      timeout        = 5000;

    /** ZooKeeper */
    protected ZooKeeper           zooKeeper;

    /**
     * 初始化
     */
    public void init() throws Exception {
        logger.info("zk connecting ......");
        zooKeeper = new ZooKeeper(server, timeout, this);
        countDownLatch.await();
        logger.info("zk connected");
        System.out.println(String.format("SessionId:0x%x", zooKeeper.getSessionId()));
    }

    /**
     * 测试方法
     */
    public void test() {
        try {
            init();
            start();
            waitForExit();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            close();
        }
    }

    /**
     * 开始
     * @throws Exception 异常
     */
    public void start() throws Exception {

    }

    /**
     * watcher
     * @param watchedEvent 事件
     */
    public void process(WatchedEvent watchedEvent) {
        System.out.println(watchedEvent);
        if (watchedEvent.getState() == Event.KeeperState.SyncConnected) {
            try {
                if (watchedEvent.getType() == Event.EventType.NodeChildrenChanged) {
                    zooKeeper.getChildren(watchedEvent.getPath(), this);
                } else if (watchedEvent.getType() == Event.EventType.NodeCreated) {
                    zooKeeper.exists(watchedEvent.getPath(), true);
                } else if (watchedEvent.getType() == Event.EventType.NodeDataChanged) {
                    Stat stat = new Stat();
                    zooKeeper.getData(watchedEvent.getPath(), this, stat);
                } else if (watchedEvent.getType() == Event.EventType.NodeDeleted) {
                    zooKeeper.exists(watchedEvent.getPath(), true);
                } else {
                    countDownLatch.countDown();
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        } else if (watchedEvent.getState() == Event.KeeperState.Disconnected) {
            countDown.countDown();
        }
    }

    /**
     * 等待退出
     */
    public void waitForExit() {
        try {
            countDown.await();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 关闭
     */
    public void close() {
        if (zooKeeper != null) {
            try {
                zooKeeper.close();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}
```
zookeeper API 同步测试
```java
package com.mydomain.zookeeper;

import java.util.List;

import org.apache.commons.lang3.StringUtils;
import org.apache.zookeeper.CreateMode;
import org.apache.zookeeper.ZooDefs;
import org.apache.zookeeper.data.Stat;

import com.mydomain.ZooTest;

/**
 * @author jyl25609
 * @version Id: ZooAPI, v 0.1 2017/4/17 15:28 jyl25609 Exp $
 */
public class ZooAPI extends ZooTest {

    /**
     * 开始
     * @throws Exception 异常
     */
    public void start() throws Exception {
        init();
        String p = "/zoo";
        String path = create(p, "123", false);
        System.out.println(path);
        p = path + "/c1";
        create(p, p, true);
        p = path + "/c2";
        create(p, p, true);
        p = path + "/c3";
        create(p, p, true);
        String value = getData(path);
        System.out.println(value);
        List<String> children = getChildren(path);
        System.out.println(StringUtils.join(children, ","));
        for (String s : children) {
            value = getData(path + "/" + s);
            System.out.println(value);
            Stat stat = setData(path + "/" + s, value + "_haha");
            System.out.println(stat);
        }
    }

    /**
     * create
     * @param path path
     * @param value value
     * @param ephemeral 是佛是临时节点
     * @return 路径
     * @throws Exception 异常
     */
    private String create(String path, String value, boolean ephemeral) throws Exception {
        delete(path);
        return zooKeeper.create(path, value.getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, ephemeral ? CreateMode.EPHEMERAL : CreateMode.PERSISTENT);
    }

    /**
     * 删除
     * @param path path
     * @throws Exception 异常
     */
    private void delete(String path) throws Exception {
        if (zooKeeper.exists(path, this) != null) {
            zooKeeper.delete(path, -1);
        }
    }

    /**
     * 获取数据
     * @param path path
     * @return 数据
     * @throws Exception 异常
     */
    private String getData(String path) throws Exception {
        Stat stat = zooKeeper.exists(path, this);
        if (stat != null) {
            byte[] data = zooKeeper.getData(path, this, stat);
            return new String(data);
        }

        return "";
    }

    /**
     * 更新数据
     * @param path path
     * @param data data
     * @return 状态
     * @throws Exception 异常
     */
    private Stat setData(String path, String data) throws Exception {
        return zooKeeper.setData(path, data.getBytes(), 0);
    }

    /**
     * 获取子节点
     * @param path path
     * @return 子节点
     * @throws Exception 异常
     */
    private List<String> getChildren(String path) throws Exception {
        return zooKeeper.getChildren(path, this);
    }
}
```
main 
```java
package com.mydomain;

import com.mydomain.zookeeper.ZooCallbackAPI;

/**
 * Hello world!
 *
 */
public class App {
    public static void main(String[] args) {
        test();
    }

    /**
     * 测试
     */
    private static void test() {
        Test zoo = new ZooAPI();
        zoo.test();
    }
}

```
## 2.2 异步方法示例   
zookeeper 异步测试
```java
package com.mydomain.zookeeper;

import java.util.concurrent.TimeUnit;

import org.apache.zookeeper.AsyncCallback;
import org.apache.zookeeper.CreateMode;
import org.apache.zookeeper.ZooDefs;
import org.apache.zookeeper.data.Stat;

import com.mydomain.ZooTest;

/**
 * @author jyl25609
 * @version Id: ZooCallbackAPI, v 0.1 2017/4/17 21:00 jyl25609 Exp $
 */
public class ZooCallbackAPI extends ZooTest {
    /**
     * 开始
     * @throws Exception 异常
     */
    public void start() throws Exception {
        init();
        String path = "/zoo/c1";
        String value = "abcdef";
        create(path, value);
        TimeUnit.SECONDS.sleep(2);
        exists(path);
        TimeUnit.SECONDS.sleep(2);
        getData(path);
        TimeUnit.SECONDS.sleep(2);
        setData(path, "hello!");
        TimeUnit.SECONDS.sleep(2);
        getData(path);
        TimeUnit.SECONDS.sleep(5);
        delete(path);
    }

    /**
     * create
     *
     * @param path path
     * @param value value
     */
    private void create(String path, String value) {
        CallbackImpl cb = new CallbackImpl();
        zooKeeper.create(path, value.getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL, cb, "create");
    }

    /**
     * delete
     *
     * @param path path
     */
    private void delete(String path) {
        VoidCallbackImpl cb = new VoidCallbackImpl();
        zooKeeper.delete(path, -1, cb, "delete");
    }

    /**
     * getData
     *
     * @param path path
     */
    private void getData(String path) {
        DataCallbackImpl cb = new DataCallbackImpl();
        zooKeeper.getData(path, this, cb, "getData");
    }

    /**
     * setData
     *
     * @param path path
     * @param value value
     */
    private void setData(String path, String value) {
        StatCallbackImpl cb = new StatCallbackImpl();
        zooKeeper.setData(path, value.getBytes(), -1, cb, "setData");
    }

    /**
     * exists
     *
     * @param path path
     */
    private void exists(String path) {
        StatCallbackImpl cb = new StatCallbackImpl();
        zooKeeper.exists(path, this, cb, "exists");
    }

    /**
     * CallbackImpl
     */
    class CallbackImpl implements AsyncCallback.StringCallback {
        public void processResult(int rc, String path, Object ctx, String name) {
            System.out.println(String.format("cxt:[%s], rc:[%s], path:[%s], name:[%s]", ctx, rc, path, name));
        }
    }

    /**
     * VoidCallbackImpl
     */
    class VoidCallbackImpl implements AsyncCallback.VoidCallback {
        public void processResult(int rc, String path, Object ctx) {
            System.out.println(String.format("cxt:[%s], rc:[%s], path:[%s]", ctx, rc, path));
        }
    }

    /**
     * DataCallbackImpl
     */
    class DataCallbackImpl implements AsyncCallback.DataCallback {
        public void processResult(int rc, String path, Object ctx, byte[] data, Stat stat) {
            System.out.println(String.format("cxt:[%s], rc:[%s], path:[%s], data:[%s], stat:[%s]", ctx, rc, path, new String(data), stat));
        }
    }

    /**
     * StatCallbackImpl
     */
    class StatCallbackImpl implements AsyncCallback.StatCallback {
        public void processResult(int rc, String path, Object ctx, Stat stat) {
            System.out.println(String.format("cxt:[%s], rc:[%s], path:[%s], stat:[%s]", ctx, rc, path, stat));
        }
    }
}
```
main
```java
package com.mydomain;

import com.mydomain.zookeeper.ZooCallbackAPI;

/**
 * Hello world!
 *
 */
public class App {
    public static void main(String[] args) {
        test();
    }

    /**
     * 测试
     */
    private static void test() {
        Test zoo = new ZooCallbackAPI();
        zoo.test();
    }
}
```

## 3. zkClient
测试类
```java
package com.mydomain.zkclient;

import java.util.Date;
import java.util.List;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

import org.I0Itec.zkclient.IZkChildListener;
import org.I0Itec.zkclient.IZkDataListener;
import org.I0Itec.zkclient.IZkStateListener;
import org.I0Itec.zkclient.ZkClient;
import org.I0Itec.zkclient.exception.ZkMarshallingError;
import org.I0Itec.zkclient.serialize.ZkSerializer;
import org.apache.commons.lang3.StringUtils;
import org.apache.zookeeper.Watcher;

import com.mydomain.Test;

/**
 * @author jyl25609
 * @version Id: ZooClientTest, v 0.1 2017/4/18 10:34 jyl25609 Exp $
 */
public class ZooClientTest implements Test {

    /** countDown */
    private static CountDownLatch countDown     = new CountDownLatch(1);

    /** 服务器 */
    private final static String   server        = "192.168.95.128:2181";

    /** ZkClient */
    private ZkClient              zkClient;

    /** IZkDataListener */
    private IZkDataListener       dataListener  = new IZkDataListenerImpl();

    /** IZkChildListener */
    private IZkChildListener      childListener = new IZkChildListenerImpl();

    /** IZkStateListener */
    private IZkStateListener      stateListener = new IZkStateListenerImpl();

    /** ZkSerializer */
    private ZkSerializer          serializer    = new ZkSerializerImpl();

    /**
     * 初始化
     */
    public void init() {
        zkClient = new ZkClient(server);
        zkClient.setZkSerializer(serializer);
        subscribeStateChanges();
    }

    /**
     * 测试方法
     */
    public void test() {
        try {
            init();
            start();
            waitForExit();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            zkClient.close();
        }
    }

    /**
     * start
     */
    public void start() throws Exception {
        String rootPath = "/zoo";
        subscribeDataChanges(rootPath);
        subscribeChildChanges(rootPath);
        String path = rootPath + "/c1";
        subscribeDataChanges(path);
        create(path, "123");
        path = rootPath + "/c2";
        subscribeDataChanges(path);
        create(path, "456");
        path = rootPath + "/c3";
        subscribeDataChanges(path);
        create(path, "789");
        List<String> children = getChildren(rootPath);
        System.out.println(StringUtils.join(children, ","));
        for (String c : children) {
            String v = getData(rootPath + "/" + c);
            System.out.println(v);
        }

        int count = 0;
        while (count++ < 200) {
            writeData(path, (new Date()).toString());
            TimeUnit.SECONDS.sleep(2);
        }
    }

    /**
     * create
     * @param path path
     * @param value value
     */
    private void create(String path, String value) {
        if (zkClient.exists(path)) {
            return;
        }

        zkClient.createEphemeral(path, value);
    }

    /**
     * getChildren
     *
     * @param path path
     * @return Children
     */
    private List<String> getChildren(String path) {
        return zkClient.getChildren(path);
    }

    /**
     * getData
     *
     * @param path path
     * @return data
     */
    private String getData(String path) {
        return zkClient.readData(path);
    }

    /**
     * writeData
     *
     * @param path path
     * @param value value
     */
    private void writeData(String path, String value) {
        zkClient.writeData(path, value);
    }

    /**
     * subscribeDataChanges
     *
     * @param path path
     */
    private void subscribeDataChanges(String path) {
        zkClient.subscribeDataChanges(path, dataListener);
    }

    /**
     * subscribeChildChanges
     *
     * @param path path
     */
    private void subscribeChildChanges(String path) {
        zkClient.subscribeChildChanges(path, childListener);
    }

    /**
     * subscribeStateChanges
     */
    private void subscribeStateChanges() {
        zkClient.subscribeStateChanges(stateListener);
    }

    /**
     * 等待退出
     */
    private void waitForExit() {
        try {
            countDown.await();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * IZkDataListenerImpl
     */
    class IZkDataListenerImpl implements IZkDataListener {
        @Override
        public void handleDataChange(String dataPath, Object data) throws Exception {
            System.out.println(String.format("[IZkDataListener][dataChange] dataPath=%s, data=%s", dataPath, data));
        }

        @Override
        public void handleDataDeleted(String dataPath) throws Exception {
            System.out.println(String.format("[IZkDataListener][dataDeleted] dataPath=%s", dataPath));
        }
    }

    /**
     * IZkChildListenerImpl
     */
    class IZkChildListenerImpl implements IZkChildListener {
        /**
         * Called when the children of the given path changed.
         *
         * @param parentPath    The parent path
         * @param currentChilds The children or null if the root node (parent path) was deleted.
         * @throws Exception Exception
         */
        @Override
        public void handleChildChange(String parentPath, List<String> currentChilds) throws Exception {
            System.out.println(String.format("[IZkChildListenerImpl][childChange] parentPath=%s, currentChilds=%s", parentPath, StringUtils.join(currentChilds, ",")));
        }
    }

    /**
     * IZkStateListenerImpl
     */
    class IZkStateListenerImpl implements IZkStateListener {
        /**
         * Called when the zookeeper connection state has changed.
         *
         * @param state The new state.
         * @throws Exception On any error.
         */
        @Override
        public void handleStateChanged(Watcher.Event.KeeperState state) throws Exception {
            System.out.println(String.format("[IZkStateListenerImpl][StateChanged] state=%s", state));
        }

        /**
         * Called after the zookeeper session has expired and a new session has been created. You would have to re-create
         * any ephemeral nodes here.
         *
         * @throws Exception On any error.
         */
        @Override
        public void handleNewSession() throws Exception {
            System.out.println("[IZkStateListenerImpl][NewSession]");
        }
    }

    /**
     * ZkSerializerImpl
     */
    class ZkSerializerImpl implements ZkSerializer {
        @Override
        public byte[] serialize(Object data) throws ZkMarshallingError {
            if (data instanceof String) {
                return String.class.cast(data).getBytes();
            }
            return new byte[0];
        }

        @Override
        public Object deserialize(byte[] bytes) throws ZkMarshallingError {
            return new String(bytes);
        }
    }
}

```
main
```java
package com.mydomain;

import com.mydomain.zkclient.ZooClientTest;

/**
 * Hello world!
 *
 */
public class App {
    public static void main(String[] args) {
        test();
    }

    /**
     * 测试
     */
    private static void test() {
        Test zoo = new ZooClientTest();
        zoo.test();
    }
}
```


## 4. Curator
### 4.1 同步方式
```java
package com.mydomain.curator;

import java.util.Date;
import java.util.List;
import java.util.concurrent.CountDownLatch;

import org.apache.commons.lang3.StringUtils;
import org.apache.curator.RetryPolicy;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.cache.*;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.apache.zookeeper.CreateMode;
import org.apache.zookeeper.data.Stat;

import com.mydomain.Test;

/**
 * @author jyl25609
 * @version Id: CuratorTest, v 0.1 2017/4/18 16:11 jyl25609 Exp $
 */
public class CuratorTest implements Test {
    /** countDown */
    private static CountDownLatch countDown = new CountDownLatch(1);

    /** 服务器 */
    private final static String   server    = "192.168.95.128:2181";

    /** CuratorFramework */
    private CuratorFramework      client;

    /**
     * 测试方法
     */
    public void test() {
        try {
            init();
            start();
            waitForExit();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            if (client != null) {
                client.close();
            }
        }
    }

    /**
     * init
     */
    private void init() {
        RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
        client = CuratorFrameworkFactory.builder().connectString(server).retryPolicy(retryPolicy).sessionTimeoutMs(30000).build();
        client.start();
    }

    /**
     * start
     *
     * @throws Exception Exception
     */
    private void start() throws Exception {
        String rootPath = "/root";

        if (null != client.checkExists().forPath(rootPath)) {
            // 如果节点存在，递归删除
            client.delete().deletingChildrenIfNeeded().forPath(rootPath);
        }

        // 创建一个空节点，默认是永久节点
        client.create().forPath(rootPath);

        // 获取值
        byte[] data = client.getData().forPath(rootPath);
        System.out.println(new String(data));

        // 删除节点
        client.delete().forPath(rootPath);
        String value = "root";

        // 创建一个值为"root"的永久节点
        client.create().forPath(rootPath, value.getBytes());

        // 节点数据变更
        dataChanged(client, rootPath);
        childrenChanged(client, rootPath);

        // 获取值
        Stat stat = new Stat();
        data = client.getData().storingStatIn(stat).forPath(rootPath);
        System.out.println(new String(data));

        // 创建临时节点
        String path = rootPath + "/" + "c1";
        client.create().withMode(CreateMode.EPHEMERAL).forPath(path, path.getBytes());
        path = rootPath + "/" + "c2";
        client.create().withMode(CreateMode.EPHEMERAL).forPath(path, path.getBytes());
        path = rootPath + "/" + "c3";
        client.create().withMode(CreateMode.EPHEMERAL).forPath(path, path.getBytes());

        // 更新数据
        client.setData().forPath(path, (new Date()).toString().getBytes());

        // 获取子节点
        List<String> children = client.getChildren().forPath(rootPath);
        System.out.println(StringUtils.join(children, ","));

        // 递归创建和删除节点
        // client.create().creatingParentsIfNeeded().forPath("", "".getBytes());
    }

    /**
     * dataChanged
     *
     * @param client client
     * @param path path
     * @throws Exception Exception
     */
    private void dataChanged(CuratorFramework client, String path) throws Exception {
        NodeCache nodeCache = new NodeCache(client, path);
        nodeCache.getListenable().addListener(new NodeCacheListenerImpl(nodeCache));
        nodeCache.start(true);
    }

    /**
     * childrenChanged
     *
     * @param client client
     * @param path path
     * @throws Exception Exception
     */
    private void childrenChanged(CuratorFramework client, String path) throws Exception {
        PathChildrenCache pathChildrenCache = new PathChildrenCache(client, path, true);
        pathChildrenCache.getListenable().addListener(new PathChildrenCacheListenerImpl());
        pathChildrenCache.start();
    }

    /**
     * 等待退出
     */
    private void waitForExit() {
        try {
            countDown.await();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * NodeCacheListenerImpl
     */
    class NodeCacheListenerImpl implements NodeCacheListener {
        /** NodeCache */
        private NodeCache nodeCache;

        /**
         * NodeCacheListenerImpl
         *
         * @param nodeCache nodeCache
         */
        public NodeCacheListenerImpl(NodeCache nodeCache) {
            this.nodeCache = nodeCache;
        }

        /**
         * nodeChanged
         *
         * @throws Exception Exception
         */
        public void nodeChanged() throws Exception {
            System.out.println(String.format("path:%s, data:%s, stat:%s", nodeCache.getCurrentData().getPath(), new String(nodeCache.getCurrentData().getData()),
                nodeCache.getCurrentData().getStat()));
        }
    }

    /**
     * PathChildrenCacheListenerImpl
     */
    class PathChildrenCacheListenerImpl implements PathChildrenCacheListener {
        /**
         * Called when a change has occurred
         *
         * @param client the client
         * @param event  describes the change
         * @throws Exception errors
         */
        public void childEvent(CuratorFramework client, PathChildrenCacheEvent event) throws Exception {
            System.out.println(event);
            System.out.println(String.format("path:%s, data:%s, stat:%s", event.getData().getPath(), new String(event.getData().getData()), event.getData().getStat()));
        }
    }
}
```
### 4.2 异步方式
```java
package com.mydomain.curator;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

import org.apache.curator.RetryPolicy;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.api.BackgroundCallback;
import org.apache.curator.framework.api.CuratorEvent;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.apache.zookeeper.CreateMode;

import com.mydomain.Test;

/**
 * @author jyl25609
 * @version Id: CuratorAsyncTest, v 0.1 2017/4/18 18:30 jyl25609 Exp $
 */
public class CuratorAsyncTest implements Test {
    /** countDown */
    private static CountDownLatch countDown = new CountDownLatch(1);

    /** 服务器 */
    private final static String   server    = "192.168.95.128:2181";

    /** CuratorFramework */
    private CuratorFramework      client;

    /**
     * 测试方法
     */
    public void test() {
        try {
            init();
            start();
            waitForExit();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            if (client != null) {
                client.close();
            }
        }
    }

    /**
     * init
     */
    private void init() {
        RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
        client = CuratorFrameworkFactory.builder().connectString(server).retryPolicy(retryPolicy).sessionTimeoutMs(30000).build();
        client.start();
    }

    /**
     * start
     *
     * @throws Exception Exception
     */
    private void start() throws Exception {
        String rootPath = "/root";
        if (null != client.checkExists().forPath(rootPath)) {
            // 如果节点存在，递归删除
            client.delete().deletingChildrenIfNeeded().forPath(rootPath);
        }

        BackgroundCallback backgroundCallback = new BackgroundCallbackImpl();
        client.create().withMode(CreateMode.EPHEMERAL).inBackground(backgroundCallback).forPath(rootPath, rootPath.getBytes());
        TimeUnit.SECONDS.sleep(2);

        client.getData().inBackground(backgroundCallback).forPath(rootPath);
    }

    /**
     * 等待退出
     */
    private void waitForExit() {
        try {
            countDown.await();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * BackgroundCallbackImpl
     */
    class BackgroundCallbackImpl implements BackgroundCallback {
        public void processResult(CuratorFramework curatorFramework, CuratorEvent curatorEvent) throws Exception {
            System.out.println(curatorEvent);
        }
    }
}
```