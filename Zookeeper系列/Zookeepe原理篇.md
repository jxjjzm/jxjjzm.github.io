### Zookeepe原理篇 ###
***

### 一、Zookeeper系统模型 ###

下面我们将从数据模型、节点特性、版本、Watcher和ACL五个方面来讲述Zookeeper的系统模型。


#### 1.数据模型 ####

Zookeeper的视图结构和标准的Unix文件系统非常类似，但没有引入传统文件系统中目录和文件等相关概念，而是使用了其特有的“数据节点”概念，我们称之为ZNode。ZNode是Zookeeper中数据的最小单元，每个ZNode上都可以保存数据，同时还可以挂载子节点，因此构成了一个层次化的命名空间，称为树。

![](http://img1.51cto.com/attachment/201211/085951545.png)

节点的路径也是使用典型的、绝对的、斜线分隔的路径样式，没有相对的引用。任何unicode字符都可以被用在路径中，除了下面的限制：

	 1）、路径名中不能包括null（\u0000）字符。（这是因为为了兼容C语言的问题）
     2）、下面的字符也不能使用，因为不能很好的显示，或者显示不正常：\u0001-\u0019和\u007F-\u009F。
     3）、下面的字符也不被允许：\ud800-uF8FFF,\uFF0-uFFFF,-uXFFFE-\uXFFFF（这里的X表示1-E）,\uF0000-\uFFFF。
     4）、点号“.”能作为路径名称的一部分，但是“.”和“..”不能单独用来表示整个路径名称，因为ZooKeeper不使用相对路径。下面的路径也是无效的：“/a/b/./c”or“/a/b/../c”。
     5）、“zookeeper”也是被保留的关键字，不允许使用。



**事务ID ———— ZXID ：**

在Zookeeper中，事务是指能够改变Zookeeper服务器状态的操作，我们也称之为事务操作或更新操作，一般包括节点创建与删除，数据节点内容更新和客户端会话创建与失效等操作。对于每一个事务请求，Zookeeper都会为其分配一个全局唯一的事务ID，用ZXID表示，通常是一个64位的数字。每一个ZXID对应一次更新操作，从这些ZXID中可以间接地识别出Zookeeper处理这些更新操作请求的全局顺序。

#### 2.节点特性 ####

在Zookeeper中，每个数据节点都是有生命周期的，其生命周期的长短取决于数据节点的节点类型。在Zookeeper中，节点类型可以分为持久节点（PERSISTENT）、临时节点（EPHEMERAL）和顺序节点（SEQUENTIAL）三大类，具体在节点创建过程中，通过组合使用，可以生成一下四种组合型节点类型：

- 持久节点(PERSISTENT)———— 所谓持久节点，是指该数据节点被创建后，就会一直存在于Zookeeper服务器上，直到有删除操作来主动清除该节点。

- 持久顺序节点(PERSISTENT_SEQUENTIAL) ———— 相比持久节点，其新增了顺序特性，每个父节点都会为它的第一级子节点维护一份顺序，用于记录每个子节点创建的先后顺序。在创建节点时，会自动添加一个数字后缀，作为一个新的、完整的节点名。需要注意的是，这个数字后缀的上限是整型的最大值。
- 临时节点（EPEMERAL）———— 临时节点的生命周期与客户端的会话绑定在一起，也就是说，如果客户端会话失效，那么这个节点会被自动清理掉。（注意，这里提到的是客户端会话失效，而非TCP连接断开。）同时，Zookeeper规定不能基于临时节点来创建子节点，即临时节点只能作为叶子节点。
- 临时顺序节点（EPEMERAL_SEQUENTIAL）———— 在临时节点的基础上添加了顺序特性。

每个数据节点除了存储了数据内容之外，还存储了数据节点本身的一些状态信息，可通过命令get获取：

	[zk: 10.1.39.43:4180(CONNECTED) 7] get /hello  
	world  
	cZxid = 0x10000042c  
	ctime = Fri May 17 17:57:33 CST 2013  
	mZxid = 0x10000042c  
	mtime = Fri May 17 17:57:33 CST 2013  
	pZxid = 0x10000042c  
	cversion = 0  
	dataVersion = 0  
	aclVersion = 0  
	ephemeralOwner = 0x0  
	dataLength = 5  
	numChildren = 0  


我们可以看到，第一行是当前数据节点的数据内容，从第二行开始就是节点的状态信息了，包括事务ID、版本信息和子节点个数等：

- czxid —— 即Created ZXID，表示该数据节点被创建时的事务ID
- mzxid —— 即Modified ZXID，表示该节点最后一次被更新时的事务ID
- ctime —— 即Created Time，表示节点被创建的时间
- mtime —— 即Modified Time，表示该节点最后一次被更新的时间
- dataVersion —— 数据节点的版本号（节点数据的更新次数）.
- cversion —— 子节点的版本号（子节点的更新次数）.
- aclVersion —— 节点的ACL版本号（节点ACL(授权信息)的更新次数）.
- ephemeralOwner —— 创建该临时节点的会话的sessionID.(如果该节点为ephemeral节点, ephemeralOwner值表示与该节点绑定的session id. 如果该节点不是ephemeral节点, ephemeralOwner值为0.)
- dataLength ——  数据内容的长度（节点数据的字节数）.
- numChildren —— 当前节点的子节点个数.
- pzxid —— 表示该节点的子节点列表最后一次被修改时的事务ID。（注意，只有子节点列表变更了才会变更pzxid,子节点内容变更不会影响pzxid）



#### 3.版本 ———— 保证分布式数据原子性操作 ####

Zookeeper中为数据节点引入了版本的概念，每个数据节点都具有三种类型的版本信息，对数据节点的任何更新操作都会引起版本号的变化。



- version ———— 当前数据节点数据内容的版本号。
- cversion ———— 当前数据节点子节点的版本号。
- aversino ———— 当前数据节点ACL变更版本号。

Zookeeper中的版本概念和传统意义上的软件版本有很大的区别，它表示的是对数据节点的数据内容、子节点列表或是节点ACL信息的修改次数。如：version为1表示对数据节点的内容变更了一次。即使前后两次变更并没有改变数据内容，version的值仍然会改变。version可以用于写入验证，类似于CAS。

![](http://img.blog.csdn.net/20160702113356844)


#### 4.Watcher ———— 数据变更的通知 ####

在Zookeeper中，引入了Watcher机制来实现这种分布式的通知功能。Zookeeper允许客户端向服务端注册一个Watcher监听，当服务端的一些指定事件触发了这个Watcher，那么就会向指定客户端发送一个事件通知来实现分布式的通知功能。

![](http://images2015.cnblogs.com/blog/616953/201611/616953-20161122094930487-367521759.png)

　Zookeeper的Watcher机制主要包括客户端线程、客户端WatcherManager、Zookeeper服务器三部分。客户端在向Zookeeper服务器注册的同时，会将Watcher对象存储在客户端的WatcherManager当中。当Zookeeper服务器触发Watcher事件后，会向客户端发送通知，客户端线程从WatcherManager中取出对应的Watcher对象来执行回调逻辑。

#### 5.ACL ———— 保障数据的安全 ####

Zookeeper内部存储了分布式系统运行时状态的元数据，这些元数据会直接影响基于Zookeeper进行构造的分布式系统的运行状态，如何保障系统中数据的安全，从而避免因误操作而带来的数据随意变更而导致的数据库异常十分重要，Zookeeper提供了一套完善的ACL权限控制机制来保障数据的安全。

　我们可以从三个方面来理解ACL机制：**权限模式（Scheme）、授权对象（ID）、权限（Permission）**，通常使用"scheme:id:permission"来标识一个有效的ACL信息。

(1)权限模式（Scheme）

权限模式用来确定权限验证过程中使用的检验策略，有如下四种模式：



- IP，通过IP地址粒度来进行权限控制，如"ip:192.168.0.110"表示权限控制针对该IP地址，同时IP模式可以支持按照网段方式进行配置，如"ip:192.168.0.1/24"表示针对192.168.0.*这个网段进行权限控制。
- Digest，使用"username:password"形式的权限标识来进行权限配置，便于区分不同应用来进行权限控制。Zookeeper会对其进行SHA-1加密和BASE64编码。
-  World，最为开放的权限控制模式，数据节点的访问权限对所有用户开放。（World模式也可以看作是一种特殊的Digest模式，它只有一个权限标识，即“World:anyone”.）
-  Super，超级用户，也是一种特殊的Digest模式，超级用户可以对任意Zookeeper上的数据节点进行任何操作。

（2）授权对象

授权对象是指权限赋予的用户或一个指定实体，如IP地址或机器等。不同的权限模式通常有不同的授权对象。

![](https://i.imgur.com/NaUhgAP.png)

（3）权限

权限是指通过权限检查可以被允许执行的操作，Zookeeper对所有数据的操作权限分为：CREATE（节点创建权限）、DELETE（节点删除权限）、READ（节点读取权限）、WRITE（节点更新权限）、ADMIN（节点管理权限）。

- CREATE（C）：数据节点的创建权限，允许授权对象在该数据节点下创建子节点。
- DELETE（D）:子节点的删除权限，允许授权对象删除该数据节点的子节点。
- READ（R）:数据节点的读取权限，允许授权对象访问该数据节点并读取其数据内容或子节点列表等。
- WRITE（W）:数据节点的更新权限，允许授权对象对该数据节点进行更新操作。
- ADMIN（A）:数据节点的管理权限，允许授权对象对该数据节点进行ACL相关的设置操作。

在绝大部分的场景下，这四种权限模式已经能够很好的实现权限控制的目的。同时，Zookeeper提供了特殊的权限控制插件体系，允许开发人员通过定制方式对Zoookeeper的权限进行自定义扩展。

### 二、Zookeeper序列化与协议 ###

#### （一）、Jute ####

Zookeeper的客户端和服务端之间会进行一系列的网络通信以实现数据的传输。对于一个网络通信，首先需要解决的就是对数据的序列化和反序列化处理，在Zookeeper中使用Jute这一序列化组件来进行数据的序列化和反序列化操作。

MockReHeader实体类：

	package com.hust.grid.leesf.jute.examples;
	
	import org.apache.jute.InputArchive;
	import org.apache.jute.OutputArchive;
	import org.apache.jute.Record;
	
	public class MockReHeader implements Record {
	    private long sessionId;
	    private String type;
	    
	    public MockReHeader() {
	        
	    }
	    
	    public MockReHeader(long sessionId, String type) {
	        this.sessionId = sessionId;
	        this.type = type;
	    }
	    
	    public void setSessionId(long sessionId) {
	        this.sessionId = sessionId;
	    }
	    
	    public void setType(String type) {
	        this.type = type;
	    }
	    
	    public long getSessionId() {
	        return sessionId;
	    }
	    
	    public String getType() {
	        return type;
	    }
	    
	    public void serialize(OutputArchive oa, String tag) throws java.io.IOException {
	        oa.startRecord(this, tag);
	        oa.writeLong(sessionId, "sessionId");
	        oa.writeString(type, "type");
	        oa.endRecord(this, tag);
	    }
	    
	    public void deserialize(InputArchive ia, String tag) throws java.io.IOException {
	        ia.startRecord(tag);
	        this.sessionId = ia.readLong("sessionId");
	        this.type = ia.readString("type");
	        ia.endRecord(tag);
	    }
	    
	    @Override
	    public String toString() {
	        return "sessionId = " + sessionId + ", type = " + type;
	    }
	}


Main：

	package com.hust.grid.leesf.jute.examples;
	
	import java.io.ByteArrayOutputStream;
	import java.io.IOException;
	import java.nio.ByteBuffer;
	
	import org.apache.jute.BinaryInputArchive;
	import org.apache.jute.BinaryOutputArchive;
	import org.apache.zookeeper.server.ByteBufferInputStream;
	
	public class Main {
	    public static void main(String[] args) throws IOException {
	        ByteArrayOutputStream baos = new ByteArrayOutputStream();
	        BinaryOutputArchive boa = BinaryOutputArchive.getArchive(baos);
	        new MockReHeader(0x3421eccb92a34el, "ping").serialize(boa, "header");
	        
	        ByteBuffer bb = ByteBuffer.wrap(baos.toByteArray());
	        
	        ByteBufferInputStream bbis = new ByteBufferInputStream(bb);
	        BinaryInputArchive bia = BinaryInputArchive.getArchive(bbis);
	        
	        MockReHeader header2 = new MockReHeader();
	        System.out.println(header2);
	        header2.deserialize(bia, "header");
	        System.out.println(header2);
	        bbis.close();
	        baos.close();    
	    }
	}

运行结果：

	sessionId = 0, type = null
	sessionId = 14673999700337486, type = ping


这个代码片段演示了如何使用Jute来对MockReqHeader对象进行序列化和反序列化，总的来说，大体可以分为4步：

	1）实体类需要实现Record接口,并实现serialize和deserialize方法。
	2）构建一个序列化器BinaryOutputArchive和反序列化器BinaryInputArchive。
	3）序列化：调用实体类的serialize方法，将对象序列化到指定tag中去。例如本例中就将MockReqHeader对象序列化到header中去。
	4）反序列化：调用实体类的deserialize,从指定的tag中反序列化出数据内容。

**1.Record接口**


	package org.apache.jute;
	import java.io.IOException;
	
	public interface Record{
		public void serialize(OutputArchive archive,String tag) throws IOException;
		public void deserialize(InputArchive archive,String tag) throws IOException;
	}

Jute定义了自己独特的序列化格式Record，Zookeeper中所有需要进行网络传输或是本地磁盘存储的类型定义都实现了该接口，其结构简单明了，操作灵活可变，是Jute序列化的核心。Record接口定义了两个最基本的方法，分别是serialize和deserialize，分别同于序列化和反序列化。其中，archive 是底层真正的序列化器和反序列化器，并且每个archive中可以包含对多个对象的序列化和反序列化，因此两个接口方法中都标记了参数tag,用于向序列化器和反序列化器标识对象自己的标记。


**2.OutputArchive和InputArchive**

OutputArchive和InputArchive分别是Jute底层的序列化器和反序列化器接口定义。在最新版本的Jute中，分别有BinaryOutputArchive/BinaryInputArchive、CsvOutputArchive/CsvInputArchive和XmlOutputArchive/XmlInputArchive三种实现。无论哪种实现，都是基于OutputStream和InputStream进行操作。



- BinaryOutputArchive/BinaryInputArchive 对数据对象的序列化和反序列化，主要用于进行网络传输和本地磁盘的存储，是Zookeeper底层最主要的序列化方式。
- CsvOutputArchive/CsvInputArchive对数据的序列化，则更多的是方便数据对象的可视化展现，因此被使用在toString方法中。
- XmlOutputArchive/XmlInputArchive 则是为了将数据对象以XML格式保存和还原，但是目前在Zookeeper中基本没有被使用到。

**3.zookeeper.jute**

很多读者在阅读Zookeeper的源码的过程中，都会发现一个有趣的现象，那就是在很多Zookeeper类的说明中，都写着“File generated by hadoop record compiler.Do not edit.” 这是因为该类并不是Zookeeper的开发人员编写的，而是通过Jute组件在编译过程中动态生成的。在Zookeeper的src文件夹下有zookeeper.jute文件，其内容如下：

	/**
	 * Licensed to the Apache Software Foundation (ASF) under one
	 * or more contributor license agreements.  See the NOTICE file
	 * distributed with this work for additional information
	 * regarding copyright ownership.  The ASF licenses this file
	 * to you under the Apache License, Version 2.0 (the
	 * "License"); you may not use this file except in compliance
	 * with the License.  You may obtain a copy of the License at
	 *
	 *     http://www.apache.org/licenses/LICENSE-2.0
	 *
	 * Unless required by applicable law or agreed to in writing, software
	 * distributed under the License is distributed on an "AS IS" BASIS,
	 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
	 * See the License for the specific language governing permissions and
	 * limitations under the License.
	 */
	
	module org.apache.zookeeper.data {
	    class Id {
	        ustring scheme;
	        ustring id;
	    }
	    class ACL {
	        int perms;
	        Id id;
	    }
	    // information shared with the client
	    class Stat {
	        long czxid;      // created zxid
	        long mzxid;      // last modified zxid
	        long ctime;      // created
	        long mtime;      // last modified
	        int version;     // version
	        int cversion;    // child version
	        int aversion;    // acl version
	        long ephemeralOwner; // owner id if ephemeral, 0 otw
	        int dataLength;  //length of the data in the node
	        int numChildren; //number of children of this node
	        long pzxid;      // last modified children
	    }
	    // information explicitly stored by the server persistently
	    class StatPersisted {
	        long czxid;      // created zxid
	        long mzxid;      // last modified zxid
	        long ctime;      // created
	        long mtime;      // last modified
	        int version;     // version
	        int cversion;    // child version
	        int aversion;    // acl version
	        long ephemeralOwner; // owner id if ephemeral, 0 otw
	        long pzxid;      // last modified children
	    }
	
	   // information explicitly stored by the version 1 database of servers 
	   class StatPersistedV1 {
	       long czxid; //created zxid
	       long mzxid; //last modified zxid
	       long ctime; //created
	       long mtime; //last modified
	       int version; //version
	       int cversion; //child version
	       int aversion; //acl version
	       long ephemeralOwner; //owner id if ephemeral. 0 otw
	    }
	}
	
	module org.apache.zookeeper.proto {
	    class ConnectRequest {
	        int protocolVersion;
	        long lastZxidSeen;
	        int timeOut;
	        long sessionId;
	        buffer passwd;
	    }
	    class ConnectResponse {
	        int protocolVersion;
	        int timeOut;
	        long sessionId;
	        buffer passwd;
	    }
	    class SetWatches {
	        long relativeZxid;
	        vector<ustring>dataWatches;
	        vector<ustring>existWatches;
	        vector<ustring>childWatches;
	    }        
	    class RequestHeader {
	        int xid;
	        int type;
	    }
	    class MultiHeader {
	        int type;
	        boolean done;
	        int err;
	    }
	    class AuthPacket {
	        int type;
	        ustring scheme;
	        buffer auth;
	    }
	    class ReplyHeader {
	        int xid;
	        long zxid;
	        int err;
	    }
	    class GetDataRequest {
	        ustring path;
	        boolean watch;
	    }
	    class SetDataRequest {
	        ustring path;
	        buffer data;
	        int version;
	    }
	    class SetDataResponse {
	        org.apache.zookeeper.data.Stat stat;
	    }
	    class GetSASLRequest {
	        buffer token;
	    }
	    class SetSASLRequest {
	        buffer token;
	    }
	    class SetSASLResponse {
	        buffer token;
	    }
	    class CreateRequest {
	        ustring path;
	        buffer data;
	        vector<org.apache.zookeeper.data.ACL> acl;
	        int flags;
	    }
	    class DeleteRequest {
	        ustring path;
	        int version;
	    }
	    class GetChildrenRequest {
	        ustring path;
	        boolean watch;
	    }
	    class GetChildren2Request {
	        ustring path;
	        boolean watch;
	    }
	    class CheckVersionRequest {
	        ustring path;
	        int version;
	    }
	    class GetMaxChildrenRequest {
	        ustring path;
	    }
	    class GetMaxChildrenResponse {
	        int max;
	    }
	    class SetMaxChildrenRequest {
	        ustring path;
	        int max;
	    }
	    class SyncRequest {
	        ustring path;
	    }
	    class SyncResponse {
	        ustring path;
	    }
	    class GetACLRequest {
	        ustring path;
	    }
	    class SetACLRequest {
	        ustring path;
	        vector<org.apache.zookeeper.data.ACL> acl;
	        int version;
	    }
	    class SetACLResponse {
	        org.apache.zookeeper.data.Stat stat;
	    }
	    class WatcherEvent {
	        int type;  // event type
	        int state; // state of the Keeper client runtime
	        ustring path;
	    }
	    class ErrorResponse {
	        int err;
	    }
	    class CreateResponse {
	        ustring path;
	    }
	    class ExistsRequest {
	        ustring path;
	        boolean watch;
	    }
	    class ExistsResponse {
	        org.apache.zookeeper.data.Stat stat;
	    }
	    class GetDataResponse {
	        buffer data;
	        org.apache.zookeeper.data.Stat stat;
	    }
	    class GetChildrenResponse {
	        vector<ustring> children;
	    }
	    class GetChildren2Response {
	        vector<ustring> children;
	        org.apache.zookeeper.data.Stat stat;
	    }
	    class GetACLResponse {
	        vector<org.apache.zookeeper.data.ACL> acl;
	        org.apache.zookeeper.data.Stat stat;
	    }
	}
	
	module org.apache.zookeeper.server.quorum {
	    class LearnerInfo {
	        long serverid;
	        int protocolVersion;
	    }
	    class QuorumPacket {
	        int type; // Request, Ack, Commit, Ping
	        long zxid;
	        buffer data; // Only significant when type is request
	        vector<org.apache.zookeeper.data.Id> authinfo;
	    }
	}
	
	module org.apache.zookeeper.server.persistence {
	    class FileHeader {
	        int magic;
	        int version;
	        long dbid;
	    }
	}
	
	module org.apache.zookeeper.txn {
	    class TxnHeader {
	        long clientId;
	        int cxid;
	        long zxid;
	        long time;
	        int type;
	    }
	    class CreateTxnV0 {
	        ustring path;
	        buffer data;
	        vector<org.apache.zookeeper.data.ACL> acl;
	        boolean ephemeral;
	    }
	    class CreateTxn {
	        ustring path;
	        buffer data;
	        vector<org.apache.zookeeper.data.ACL> acl;
	        boolean ephemeral;
	        int parentCVersion;
	    }
	    class DeleteTxn {
	        ustring path;
	    }
	    class SetDataTxn {
	        ustring path;
	        buffer data;
	        int version;
	    }
	    class CheckVersionTxn {
	        ustring path;
	        int version;
	    }
	    class SetACLTxn {
	        ustring path;
	        vector<org.apache.zookeeper.data.ACL> acl;
	        int version;
	    }
	    class SetMaxChildrenTxn {
	        ustring path;
	        int max;
	    }
	    class CreateSessionTxn {
	        int timeOut;
	    }
	    class ErrorTxn {
	        int err;
	    }
	    class Txn {
	        int type;
	        buffer data;
	    }
	    class MultiTxn {
	        vector<org.apache.zookeeper.txn.Txn> txns;
	    }
	}

其定义了所有的实体类的所属包名、类名及类的所有成员变量和类型，该文件会在源代码编译时，Jute会使用不同的代码生成器为这些类定义生成实际编程语言的类文件。如java语言，Jute会使用JavaGenerator来生成相应的类文件，生成的类文件保存在src/java/generated目录下。需要注意的一点是，使用这种方式生成的类都会实现Record接口。


#### （二）、通信协议 ####

基于TCP/IP协议，Zookeeper实现了自己的通信协议来完成客户端与服务端、服务端与服务端之间的网络通信。对于请求，主要包含请求头和请求体；对于响应，主要包含响应头和响应体。

![](http://images2015.cnblogs.com/blog/616953/201611/616953-20161124095349737-1635431714.png)

**1.协议解析：请求部分**

![](https://i.imgur.com/DxO6eKj.png)

**I、请求头**

	 class RequestHeader {
	        int xid;
	        int type;
	    }

从zookeeper.jute中可知RequestHeader包含了xid和type，xid用于记录客户端请求发起的先后序号，用来确保单个客户端请求的响应顺序，type代表请求的操作类型，如创建节点（OpCode.create）、删除节点（OpCode.delete）、获取节点数据（OpCode.getData）。　根据协议规定，除非是“会话创建”请求，其他所有客户端请求中都会带上请求头。

**II、请求体**

协议的请求体部分是指请求的主体内容部分，包含了请求的所有操作内容。不同的请求类型，其请求体部分的结构是不同的。

（1）ConnectRequest：会话创建

	class ConnectRequest {
	        int protocolVersion;
	        long lastZxidSeen;
	        int timeOut;
	        long sessionId;
	        buffer passwd;
	    }

Zookeeper客户端和服务器在创建会话时，会发送ConnectRequest请求，该请求包含协议版本号protocolVersion、最近一次接收到服务器ZXID lastZxidSeen、会话超时时间timeOut、会话标识sessionId和会话密码passwd。


（2）GetDataRequest：获取节点数据

	 class GetDataRequest {
	        ustring path;
	        boolean watch;
	    }

Zookeeper客户端在向服务器发送节点数据请求时，会发送GetDataRequest请求，该请求包含了数据节点路径path、是否注册Watcher的标识watch。


（3）SetDataRequest：更新节点数据


	 class SetDataRequest {
	        ustring path;
	        buffer data;
	        int version;
	    }


Zookeeper客户端在向服务器发送更新节点数据请求时，会发送SetDataRequest请求，该请求包含了数据节点路径path、数据内容data、节点数据的期望版本号version。


**2.协议解析：响应部分**

![](https://i.imgur.com/Fe09h7S.png)

**I、响应头**

	class ReplyHeader {
	        int xid;
	        long zxid;
	        int err;
	    }

　响应头中包含了每个响应最基本的信息，包括xid、zxid和err：　　xid与请求头中的xid一致，zxid表示Zookeeper服务器上当前最新的事务ID，err则是一个错误码，表示当请求处理过程出现异常情况时，就会在错误码中标识出来，常见的包括处理成功（Code.OK：0）、节点不存在（Code.NONODE：101）、没有权限（Code.NOAUTH：102）等，所有这些错误码都被定义在类org.apache.zookeeper.KeeperException.Code中。


**II、响应体**

协议的响应体部分是指响应的主体内容部分，包含了响应的所有返回数据。不同的响应类型，其响应体部分的结构是不同的。

（1）ConnectResponse：会话创建

	class ConnectResponse {
	        int protocolVersion;
	        int timeOut;
	        long sessionId;
	        buffer passwd;
	    }

针对客户端的会话创建请求，服务端会返回客户端一个ConnectResponse响应，该响应体包含了版本号protocolVersion、会话的超时时间timeOut、会话标识sessionId和会话密码passwd。

（2）GetDataResponse：获取节点数据

	class GetDataResponse {
	        buffer data;
	        org.apache.zookeeper.data.Stat stat;
	    }

针对客户端的获取节点数据请求，服务端会返回客户端一个GetDataResponse响应，该响应体包含了数据节点内容data、节点状态stat。



（3）SetDataResponse：更新节点数据

	class SetDataResponse {
	        org.apache.zookeeper.data.Stat stat;
	    }

　针对客户端的更新节点数据请求，服务端会返回客户端一个SetDataResponse响应，该响应体包含了最新的节点状态stat。


（详细可以在zookeeper.jute中查看。）


### 三、Zookeeper客户端 ###

客户端是开发人员使用Zookeeper最主要的途径，因此我们有必要对Zookeeper客户端的内部原理进行详细讲解。Zookeeper的客户端主要由以下几个核心组件组成。



- Zookeeper实例:客户端入口。
- ClientWatchManager： 客户端Watcher管理器。
- HostProvider，客户端地址列表管理器。
-  ClientCnxn，客户端核心线程，内部包含了SendThread和EventThread两个线程，SendThread为I/O线程，主要负责Zookeeper客户端和服务器之间的网络I/O通信；EventThread为事件线程，主要负责对服务端事件进行处理。

![](http://images2015.cnblogs.com/blog/616953/201611/616953-20161124170014471-1163146559.png)

Zookeeper客户端初始化与启动环节，实际就是Zookeeper对象的实例化过程。客户端的整个初始化和启动过程大体可以分为如下3个步骤：



- 设置默认Watcher
- 设置Zookeeper服务器地址列表
- 创建ClientCnxn。

　　若在Zookeeper构造方法中传入Watcher对象时，那么Zookeeper就会将该Watcher对象保存在ZKWatcherManager的defaultWatcher中，并作为整个客户端会话期间的默认Watcher。

#### 1.一次会话的创建过程 ####

![](http://images2015.cnblogs.com/blog/616953/201611/616953-20161124205059237-613598814.png)



**I、初始化阶段**


- 初始化Zookeeper对象：通过调用Zookeeper的构造方法来实例化一个Zookeeper对象，在初始化过程中，会创建一个客户端的Watcher管理ClientWatchManager。
- 设置会话默认Watcher：如果在构造方法中传入了一个Watcher对象，那么客户端会将这个对象作为默认Watcher保存在ClinetWatchManager中。
- 构造Zookeeper服务器地址列表管理器HostProvider:对于构造方法中传入的服务器地址，客户端会将其存放在服务器地址列表管理器HostProvider中。
- 创建并初始化客户端网络连接器ClientCnxn：Zookeeper客户端首先会创建一个网络连接器ClientCnxn，用来管理客户端与服务器的网络交互。另外，客户端在创建ClientCnxn的同时，还会初始化客户端两个核心队列outgoingQueue和pendingQueue,分别作为客户端的请求发送队列和服务端响应的等待队列。
- 初始化SendThread和EventThread ：客户端会创建两个核心网络线程SendThread和EventThread，前者用于管理客户端和服务端之间的所有网络IO，后者则用于进行客户端的事件处理。同时，客户端还会将ClientCnxnSocket分配给SendThread作为底层网络IO处理器，并初始化EventThread的待处理事件队列waitingEvents,用于存放所有等待被客户端处理的事件。


**II、会话创建阶段**

- 启动SendThread和EventThread：SendThread首先会判断当前客户端的状态，进行一系列清理性工作，为客户端发送“会话创建”请求做准备。
- 获取一个服务端地址： 在开始创建TCP连接之前，SendThread首先需要获取一个Zookeeper服务器的目标地址，这通常是从HostProvider中随机获取出一个地址，然后委托给ClientCnxnSocket去创建与Zookeeper服务器之间的TCP连接。
- 创建TCP连接：获取到一个服务器地址后，ClientCnxnSocket负责和服务器创建一个TCP长连接。
- 构造ConnnectRequest请求：SendThread会负责根据当前客户端的实际设置，构造出一个ConnectRequest请求，该请求代表了客户端试图与服务器创建一个会话。同时，Zookeeper客户端还会进一步将该请求包装成网络IO层的Packet对象，放入请求发送队列outgoingQueue中去。
- 发送请求：当客户端请求准备完毕后，就可以开始向服务器发送请求了。ClientCnxnSocket负责从outgoingQueue中取出一个待发送的Packet对象，将其序列化成byteBuffer后，向服务端进行发送。



**III、响应处理阶段**



- 接收服务端响应：ClientCnxnSocket接收到服务端的响应后，会首先判断当前的客户端状态是否是“已初始化”，如果尚未完成初始化，那么就认为该响应一定是会话创建请求的响应，直接交由readConnectResult方法来处理该响应。
- 处理Response:ClinetCnxnSocket会对接收到的服务端响应进行反序列化，得到ConnectResponse对象，并从中获取到Zookeeper服务端分配的会话SeesionId。
- 连接成功：连接成功后，一方面需要通知SendThread线程，进一步对客户端进行会话参数的设置，包括readTimeout和connectTimeout等，并更新客户端状态；另一方面，需要通知地址管理器HostProvider当前成功连接的服务器地址。
- 生成事件：SynConnected-None：为了能够让上层应用感知到会话的成功创建，SendThread会生成一个事件SyncConnected-None，代表客户端与服务器会话创建成功，并将该事件传递给EventThread线程。
- 查询Watcher：EventThread线程收到事件后，慧聪ClientWatchManager管理器中查询出对应的Watcher，针对SyncConnected-None事件，那么就直接找出步骤2中存储的默认Watcher，然后将其放到EventThread的waitingEvents队列中去。
- 处理时间：EventThread不断地从waitingEvents队列中取出待处理的Watcher对象，然后直接调用该对象的process接口方法，以达到触发Watcher的目的。

至此，Zookeeper客户端完整的一次会话创建过程已经全部完成了

#### 2.服务器地址列表 ####

在实例化Zookeeper时，用户传入Zookeeper服务器地址列表，如192.168.0.1:2181,192.168.0.2:2181,192.168.0.3:2181，此时，Zookeeper客户端在连接服务器的过程中，是如何从这个服务器列表中选择服务器的呢？Zookeeper收到服务器地址列表后，会解析出chrootPath和保存服务器地址列表。

- Chroot，每个客户端可以设置自己的命名空间，若客户端设置了Chroot，此时，该客户端对服务器的任何操作都将被限制在自己的命名空间下，如设置Choot为/app/X，那么该客户端的所有节点路径都是以/app/X为根节点。

-  地址列表管理，Zookeeper使用StaticHostProvider打散服务器地址（shuffle），并将服务器地址形成一个环形循环队列，然后再依次取出服务器地址。

#### 3.ClientCnxn：网络IO ####

ClientCnxn是Zookeeper客户端中负责维护客户端与服务端之间的网络连接并进行一系列网络通信的核心工作类。

**I、Packet**

Packet是ClientCnxn内部定义的一个堆协议层的封装，用作Zookeeper中请求和响应的载体。Packet包含了请求头（requestHeader）、响应头（replyHeader）、请求体（request）、响应体（response）、节点路径（clientPath/serverPath）、注册的Watcher（watchRegistration）等信息，然而，并非Packet中所有的属性都在客户端与服务端之间进行网络传输，只会将requestHeader、request、readOnly三个属性序列化，并生成可用于底层网络传输的ByteBuffer，其他属性都保存在客户端的上下文中，不会进行与服务端之间的网络传输。

**II、outgoingQueue 和 pendingQueue**

ClientCnxn维护着outgoingQueue（客户端的请求发送队列）和pendingQueue（服务端响应的等待队列），outgoingQueue专门用于存储那些需要发送到服务端的Packet集合，pendingQueue用于存储那些已经从客户端发送到服务端的，但是需要等待服务端响应的Packet集合。

![](http://images2015.cnblogs.com/blog/616953/201611/616953-20161124211747487-571974849.png)

在正常情况下，会从outgoingQueue中取出一个可发送的Packet对象，同时生成一个客户端请求序号XID并将其设置到Packet请求头中去，然后序列化后再发送，请求发送完毕后，会立即将该Packet保存到pendingQueue中，以便等待服务端响应返回后进行相应的处理。客户端获取到来自服务端的完整响应数据后，根据不同的客户端请求类型，会进行不同的处理:

-  若检测到此时客户端尚未进行初始化，那么说明当前客户端与服务端之间正在进行会话创建，直接将接收的ByteBuffer序列化成ConnectResponse对象。
-  若当前客户端已经处于正常会话周期，并且接收到服务端响应是一个事件，那么将接收的ByteBuffer序列化成WatcherEvent对象，并将该事件放入待处理队列中。
-   若是一个常规请求（Create、GetData、Exist等），那么从pendingQueue队列中取出一个Packet来进行相应处理。首先会检验响应中的XID来确保请求处理的顺序性，然后再将接收到的ByteBuffer序列化成Response对象。

**III、SendThread**

SendThread是客户端ClientCnxn内部的一个核心I/O调度线程，用于管理客户端与服务端之间的所有网络I/O操作，在Zookeeper客户端实际运行中，SendThread的作用如下：


- 维护了客户端与服务端之间的会话生命周期（通过一定周期频率内向服务端发送PING包检测心跳），如果会话周期内客户端与服务端出现TCP连接断开，那么就会自动且透明地完成重连操作。
- 管理了客户端所有的请求发送和响应接收操作，其将上层客户端API操作转换成相应的请求协议并发送到服务端，并完成对同步调用的返回和异步调用的回调。
- 将来自服务端的事件传递给EventThread去处理。

**IV、EventThread**

EventThread是客户端ClientCnxn内部的一个事件处理线程，负责客户端的事件处理，并触发客户端注册的Watcher监听。EventThread中的watingEvents队列用于临时存放那些需要被触发的Object，包括客户端注册的Watcher和异步接口中注册的回调器AsyncCallback。同时，EventThread会不断地从watingEvents中取出Object，识别具体类型（Watcher或AsyncCallback），并分别调用process和processResult接口方法来实现对事件的触发和回调。



### 四、Zookeeper会话 ###

客户端与服务端之间任何交互操作都与会话息息相关，如临时节点的生命周期、客户端请求的顺序执行、Watcher通知机制等。Zookeeper的连接与会话就是客户端通过实例化Zookeeper对象来实现客户端与服务端创建并保持TCP连接的过程.

#### 1.会话状态 ####

![](http://images2015.cnblogs.com/blog/616953/201611/616953-20161126160605690-2108890253.png)

在Zookeeper客户端与服务端成功完成连接创建后，就创建了一个会话，Zookeeper会话在整个运行期间的生命周期中，会在不同的会话状态中之间进行切换，这些状态可以分为CONNECTING、CONNECTED、RECONNECTING、RECONNECTED、CLOSE等。

一旦客户端开始创建Zookeeper对象，那么客户端状态就会变成CONNECTING状态，同时客户端开始尝试连接服务端，连接成功后，客户端状态变为CONNECTED，通常情况下，由于断网或其他原因，客户端与服务端之间会出现断开情况，一旦碰到这种情况，Zookeeper客户端会自动进行重连服务，同时客户端状态再次变成CONNCTING，直到重新连上服务端后，状态又变为CONNECTED，在通常情况下，客户端的状态总是介于CONNECTING和CONNECTED之间。但是，如果出现诸如会话超时、权限检查或是客户端主动退出程序等情况，客户端的状态就会直接变更为CLOSE状态。


#### 2.会话创建 ####

Session是Zookeeper中的会话实体，代表了一个客户端会话，其包含了如下四个属性：

- sessionID。会话ID，唯一标识一个会话，每次客户端创建新的会话时，Zookeeper都会为其分配一个全局唯一的sessionID。
- TimeOut。会话超时时间，客户端在构造Zookeeper实例时，会配置sessionTimeout参数用于指定会话的超时时间，Zookeeper客户端向服务端发送这个超时时间后，服务端会根据自己的超时时间限制最终确定会话的超时时间。
- TickTime。下次会话超时时间点，为了便于Zookeeper对会话实行"分桶策略"管理，同时为了高效低耗地实现会话的超时检查与清理，Zookeeper会为每个会话标记一个下次会话超时时间点，其值大致等于当前时间加上TimeOut。
- isClosing。标记一个会话是否已经被关闭，当服务端检测到会话已经超时失效时，会将该会话的isClosing标记为"已关闭"，这样就能确保不再处理来自该会话的心情求了。

#### 3.会话管理 ####

**I、分桶策略**

Zookeeper的会话管理主要是通过SessionTracker来负责，其采用了分桶策略（将类似的会话放在同一区块中进行管理）进行管理，以便Zookeeper对会话进行不同区块的隔离处理以及同一区块的统一处理。

![](http://images2015.cnblogs.com/blog/616953/201611/616953-20161126165451643-351272465.png)

Zookeeper将所有的会话都分配在不同的区块一种，分配的原则是每个会话的下次超时时间点（ExpirationTime）。ExpirationTime指该会话最近一次可能超时的时间点。同时，Zookeeper Leader服务器在运行过程中会定时地进行会话超时检查，时间间隔是ExpirationInterval，默认为tickTime的值，ExpirationTime的计算时间如下

　　ExpirationTime = ((CurrentTime + SessionTimeOut) / ExpirationInterval + 1) * ExpirationInterval


**II、会话激活**

为了保持客户端会话的有效性，客户端会在会话超时时间过期范围内向服务端发送PING请求来保持会话的有效性（心跳检测）。同时，服务端需要不断地接收来自客户端的心跳检测，并且需要重新激活对应的客户端会话，这个重新激活过程称为TouchSession。会话激活不仅能够使服务端检测到对应客户端的存货性，同时也能让客户端自己保持连接状态，其流程如下：　

![](http://images2015.cnblogs.com/blog/616953/201611/616953-20161126171119628-1511238863.png)

在上面会话激活过程中，只要客户端发送心跳检测，服务端就会进行一次会话激活，心跳检测由客户端主动发起，以PING请求形式向服务端发送，在Zookeeper的实际设计中，只要客户端有请求发送到服务端，那么就会触发一次会话激活，以下两种情况都会触发会话激活：

- 客户端向服务端发送请求，包括读写请求，就会触发会话激活。
-  客户端发现在sessionTimeout/3时间内尚未和服务端进行任何通信，那么就会主动发起PING请求，服务端收到该请求后，就会触发会话激活。

**III、会话超时检查**

对于会话的超时检查而言，Zookeeper使用SessionTracker来负责，SessionTracker使用单独的线程（超时检查线程）专门进行会话超时检查，即逐个一次地对会话桶中剩下的会话进行清理。如果一个会话被激活，那么Zookeeper就会将其从上一个会话桶迁移到下一个会话桶中，如session.n 这个会话由于触发了会话激活，因此Zookeeper会将其从ExpirationTime 1桶迁移到ExpirationTime n 桶中去。于是ExpirationTime 1中留下的所有会话都是尚未被激活的，超时检查线程就定时检查这个会话桶中所有剩下的未被迁移的会话，超时检查线程只需要在这些指定时间点（ExpirationTime 1、ExpirationTime 2...）上进行检查即可，这样提高了检查的效率，性能也非常好。

**IV、会话清理**

当SessionTracker的会话超时线程检查出已经过期的会话后，就开始进行会话清理工作，大致可以分为如下七步。

	　　1. 标记会话状态为已关闭。由于会话清理过程需要一段时间，为了保证在此期间不再处理来自该客户端的请求，SessionTracker会首先将该会话的isClosing标记为true，这样在会话清理期间接收到该客户端的心情求也无法继续处理了。
	
	　　2. 发起会话关闭请求。为了使对该会话的关闭操作在整个服务端集群都生效，Zookeeper使用了提交会话关闭请求的方式，并立即交付给PreRequestProcessor进行处理。
	
	　　3. 收集需要清理的临时节点。一旦某个会话失效后，那么和该会话相关的临时节点都需要被清理，因此，在清理之前，首先需要将服务器上所有和该会话相关的临时节点都整理出来。Zookeeper在内存数据库中会为每个会话都单独保存了一份由该会话维护的所有临时节点集合，在Zookeeper处理会话关闭请求之前，若正好有以下两类请求到达了服务端并正在处理中。
	
	　　　　· 节点删除请求，删除的目标节点正好是上述临时节点中的一个。
	
	　　　　· 临时节点创建请求，创建的目标节点正好是上述临时节点中的一个。
	
	　　对于第一类请求，需要将所有请求对应的数据节点路径从当前临时节点列表中移出，以避免重复删除，对于第二类请求，需要将所有这些请求对应的数据节点路径添加到当前临时节点列表中，以删除这些即将被创建但是尚未保存到内存数据库中的临时节点。
	
	　　4. 添加节点删除事务变更。完成该会话相关的临时节点收集后，Zookeeper会逐个将这些临时节点转换成"节点删除"请求，并放入事务变更队列outstandingChanges中。
	
	　　5. 删除临时节点。FinalRequestProcessor会触发内存数据库，删除该会话对应的所有临时节点。
	
	　　6. 移除会话。完成节点删除后，需要将会话从SessionTracker中删除。
	
	　　7. 关闭NIOServerCnxn。最后，从NIOServerCnxnFactory找到该会话对应的NIOServerCnxn，将其关闭。


**V、重连**

当客户端与服务端之间的网络连接断开时，Zookeeper客户端会自动进行反复的重连，直到最终成功连接上Zookeeper集群中的一台机器。此时，再次连接上服务端的客户端有可能处于以下两种状态之一

　　

-  CONNECTED。如果在会话超时时间内重新连接上集群中一台服务器 。
-  EXPIRED。如果在会话超时时间以外重新连接上，那么服务端其实已经对该会话进行了会话清理操作，此时会话被视为非法会话。

在客户端与服务端之间维持的是一个长连接，在sessionTimeout时间内，服务端会不断地检测该客户端是否还处于正常连接，服务端会将客户端的每次操作视为一次有效的心跳检测来反复地进行会话激活。因此，在正常情况下，客户端会话时一直有效的。然而，当客户端与服务端之间的连接断开后，用户在客户端可能主要看到两类异常：CONNECTION_LOSS（连接断开）和SESSION_EXPIRED（会话过期）。

- CONNECTION_LOSS。此时，客户端会自动从地址列表中重新逐个选取新的地址并尝试进行重新连接，直到最终成功连接上服务器。若客户端在setData时出现了CONNECTION_LOSS现象，此时客户端会收到None-Disconnected通知，同时会抛出异常。应用程序需要捕捉异常并且等待Zookeeper客户端自动完成重连，一旦重连成功，那么客户端会收到None-SyncConnected通知，之后就可以重试setData操作。
-  SESSION_EXPIRED。客户端与服务端断开连接后，重连时间耗时太长，超过了会话超时时间限制后没有成功连上服务器，服务器会进行会话清理，此时，客户端不知道会话已经失效，状态还是DISCONNECTED，如果客户端重新连上了服务器，此时状态为SESSION_EXPIRED，用于需要重新实例化Zookeeper对象，并且看应用的复杂情况，重新恢复临时数据。
-  SESSION_MOVED。客户端会话从一台服务器转移到另一台服务器，即客户端与服务端S1断开连接后，重连上了服务端S2，此时会话就从S1转移到了S2。当多个客户端使用相同的sessionId/sessionPasswd创建会话时，会收到SessionMovedException异常。因为一旦有第二个客户端连接上了服务端，就被认为是会话转移了。


### 五、Zookeeper服务端启动 ###

服务端整体架构如下：

![](http://images2015.cnblogs.com/blog/616953/201611/616953-20161126210233253-1268373589.png)

Zookeeper服务器的启动，大致可以分为以下五个步骤：配置文件解析、初始化数据管理器、初始化网络IO管理器、数据恢复和对外服务。

#### 1.单机版服务器启动 ####

![](http://images2015.cnblogs.com/blog/616953/201611/616953-20161127105308893-475081399.png)


**I. 预启动阶段**　　　

- 1）统一由QuorumPeerMain作为启动类。————无论单机或集群，在zkServer.cmd和zkServer.sh中都配置了QuorumPeerMain作为启动入口类。
- 2）解析配置文件zoo.cfg。————zoo.cfg配置运行时的基本参数，如tickTime、dataDir、clientPort等参数。
- 3）创建并启动历史文件清理器DatadirCleanupManager————对事务日志和快照数据文件进行定时清理。
- 4） 判断当前是集群模式还是单机模式启动。若是单机模式，则委托给ZooKeeperServerMain进行启动。
- 5） 再次进行配置文件zoo.cfg的解析。
- 6） 创建服务器实例ZooKeeperServer。————Zookeeper服务器首先会进行服务器实例的创建，然后对该服务器实例进行初始化，包括连接器、内存数据库、请求处理器等组件的初始化。


**II. 初始化阶段**

　　　

- 1）创建服务器统计器ServerStats。————ServerStats是Zookeeper服务器运行时的统计器。
-  2）创建Zookeeper数据管理器FileTxnSnapLog。————FileTxnSnapLog是Zookeeper上层服务器和底层数据存储之间的对接层，提供了一系列操作数据文件的接口，如事务日志文件和快照数据文件。Zookeeper根据zoo.cfg文件中解析出的快照数据目录dataDir和事务日志目录dataLogDir来创建FileTxnSnapLog。
-  3）设置服务器tickTime和会话超时时间限制。
-  4）创建ServerCnxnFactory。————通过配置系统属性zookeper.serverCnxnFactory来指定使用Zookeeper自己实现的NIO还是使用Netty框架作为Zookeeper服务端网络连接工厂。
-  5）初始化ServerCnxnFactory。————Zookeeper首先会初始化Thread作为ServerCnxnFactory的主线程，然后再初始化NIO服务器。
-  6）启动ServerCnxnFactory主线程。————进入Thread的run方法，此时服务端还不能处理客户端请求。
-  7） 恢复本地数据。启动时，需要从本地快照数据文件和事务日志文件进行数据恢复。
-  8） 创建并启动会话管理器。————Zookeeper会创建会话管理器SessionTracker进行会话管理。
-  9） 初始化Zookeeper的请求处理链。————Zookeeper请求处理方式为责任链模式的实现。会有多个请求处理器依次处理一个客户端请求，在服务器启动时，会将这些请求处理器串联成一个请求处理链。
-  10）  注册JMX服务。Zookeeper会将服务器运行时的一些信息以JMX的方式暴露给外部。
-  11）  注册Zookeeper服务器实例。将Zookeeper服务器实例注册给ServerCnxnFactory，之后Zookeeper就可以对外提供服务。

至此，单机版的Zookeeper服务器启动完毕。



#### 2.集群版服务器启动 ####

![](http://images2015.cnblogs.com/blog/616953/201611/616953-20161127110817253-53261436.png)

**I、预启动阶段**

- 1）统一由QuorumPeerMain作为启动类。
- 2）解析配置文件zoo.cfg。
- 3）创建并启动历史文件清理器DatadirCleanupFactory。
- 4）判断当前是集群模式还是单机模式的启动。在集群模式中，由于已经在zoo.cfg文件中配置了多个服务器地址，因此此处选择集群模式启动Zookeeper。


**II、初始化阶段**

- 1）创建ServerCnxnFactory。
- 2）初始化ServerCnxnFactory。
- 3）创建Zookeeper数据管理器FileTxnSnapLog。
- 4）创建QuorumPeer实例。————Quorum是集群模式下特有的对象，是Zookeeper服务器实例（ZooKeeperServer）的托管者，QuorumPeer代表了集群中的一台机器，在运行期间，QuorumPeer会不断检测当前服务器实例的运行状态，同时根据情况发起Leader选举。
- 5）创建内存数据库ZKDatabase。————ZKDatabase负责管理ZooKeeper的所有会话记录以及DataTree和事务日志的存储。
- 6） 初始化QuorumPeer。————将核心组件如FileTxnSnapLog、ServerCnxnFactory、ZKDatabase注册到QuorumPeer中，同时配置QuorumPeer的参数，如服务器列表地址、Leader选举算法和会话超时时间限制等。
- 7）恢复本地数据。
- 8）启动ServerCnxnFactory主线程。


**III、Leader选举阶段**

- 1）初始化Leader选举。————集群模式特有，Zookeeper首先会根据自身的服务器ID（SID）、最新的ZXID（lastLoggedZxid）和当前的服务器epoch（currentEpoch）来生成一个初始化投票，在初始化过程中，每个服务器都会给自己投票。然后，根据zoo.cfg的配置，创建相应Leader选举算法实现，Zookeeper提供了三种默认算法（LeaderElection、AuthFastLeaderElection、FastLeaderElection），可通过zoo.cfg中的electionAlg属性来指定，但从3.4.0版本开始Zokeeper废弃了前两种Leader选举算法，只支持FastLeaderElection选举算法了。在初始化阶段，Zookeeper会创建Leader选举所需的网络I/O层QuorumCnxManager，同时启动对Leader选举端口的监听，等待集群中其他服务器创建连接。
- 2）注册JMX服务。
- 3）检测当前服务器状态。————运行期间，QuorumPeer会不断检测当前服务器状态。在正常情况下，Zookeeper服务器的状态在LOOKING、LEADING、FOLLOWING/OBSERVING之间进行切换。在启动阶段，QuorumPeer的初始状态是LOOKING，因此开始进行Leader选举。
- 4）Leader选举。————Zookeeper的Leader选举过程，简单来讲，就是一个集群中所有的机器相互之间进行一系列投票，选举产生最合适的机器成为Leader，同时其余机器成为Follower或是Observer的集群机器角色初始化过程。关于Leader选举算法，简而言之，就是集群中哪个机器处理的数据越新（通常我们根据每个服务器处理过的最大ZXID来比较确定其数据是否更新），其越有可能成为Leader。当然，如果集群中的所有机器处理的ZXID一致的话，那么SID最大的服务器成为Leader。具体算法在后面会给出详细讲解。


**IV、Leader和Follower启动期交互过程**

- 1）创建Leader服务器和Follower服务器。————完成Leader选举后，每个服务器会根据自己服务器的角色创建相应的服务器实例，并进入各自角色的主流程。
- 2）Leader服务器启动Follower接收器LearnerCnxAcceptor。————在Zookeeper集群运行期间，Leader服务器需要和所有其余的服务器（统称为Learner）保持连接以确定集群的机器存活情况，LearnerCnxAcceptor负责接收所有非Leader服务器的连接请求。
- 3）Leader服务器开始和Leader建立连接。————所有Learner会找到Leader服务器，并与其建立连接。
- 4） Leader服务器创建LearnerHandler。————Leader接收到来自其他机器连接创建请求后，会创建一个LearnerHandler实例，每个LearnerHandler实例都对应一个Leader与Learner服务器之间的连接，其负责Leader和Learner服务器之间几乎所有的消息通信和数据同步。
- 5）向Leader注册。————Learner完成和Leader的连接后，会向Leader进行注册，即将Learner服务器的基本信息（LearnerInfo），包括SID和ZXID，发送给Leader服务器。
- 6） Leader解析Learner信息，计算新的epoch。————Leader接收到Learner服务器基本信息后，会解析出该Learner的SID和ZXID，然后根据ZXID解析出对应的epoch_of_learner，并和当前Leader服务器的epoch_of_leader进行比较，如果该Learner的epoch_of_learner更大，则更新Leader的epoch_of_leader = epoch_of_learner + 1。然后LearnHandler进行等待，直到过半Learner已经向Leader进行了注册，同时更新了epoch_of_leader后，Leader就可以确定当前集群的epoch了。
- 7） 发送Leader状态。————计算出新的epoch后，Leader会将该信息以一个LEADERINFO消息的形式发送给Learner，并等待Learner的响应。
- 8） Learner发送ACK消息。————Learner接收到LEADERINFO后，会解析出epoch和ZXID，然后向Leader反馈一个ACKEPOCH响应。
- 9） 数据同步。————Leader收到Learner的ACKEPOCH后，即可进行数据同步。
- 10）启动Leader和Learner服务器。————当有过半Learner已经完成了数据同步，那么Leader和Learner服务器实例就可以启动了。

**V、Leader和Follower启动**

- 1）创建启动会话管理器。
- 2）初始化Zookeeper请求处理链，集群模式的每个处理器也会在启动阶段串联请求处理链，只是根据服务器角色不同，会有不同的请求处理链路。
- 3）注册JMX服务。

至此，集群版的Zookeeper服务器启动完毕。




### 六、Zookeeper Leader选举 ###







### 七、Zookeeper各服务器角色介绍 ###







### 八、Zookeeper请求处理 ###








### 九、Zookeeper数据与存储 ###



















