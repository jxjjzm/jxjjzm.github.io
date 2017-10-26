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

#### 1.Leader选举概述 ####

**I、服务器启动时期的Leader选举**

若进行Leader选举，则至少需要两台机器，这里选取3台机器组成的服务器集群为例。在集群初始化阶段，当有一台服务器Server1启动时，其单独无法进行和完成Leader选举，当第二台服务器Server2启动时，此时两台机器可以相互通信，每台机器都试图找到Leader，于是进入Leader选举过程。选举过程如下:

	　　(1) 每个Server发出一个投票。由于是初始情况，Server1和Server2都会将自己作为Leader服务器来进行投票，每次投票会包含所推举的服务器的myid和ZXID，使用(myid, ZXID)来表示，此时Server1的投票为(1, 0)，Server2的投票为(2, 0)，然后各自将这个投票发给集群中其他机器。
	
	　　(2) 接受来自各个服务器的投票。集群的每个服务器收到投票后，首先判断该投票的有效性，如检查是否是本轮投票、是否来自LOOKING状态的服务器。
	
	　　(3) 处理投票。针对每一个投票，服务器都需要将别人的投票和自己的投票进行PK，PK规则如下
	
	　　　　· 优先检查ZXID。ZXID比较大的服务器优先作为Leader。
	
	　　　　· 如果ZXID相同，那么就比较myid。myid较大的服务器作为Leader服务器。
	
	　　对于Server1而言，它的投票是(1, 0)，接收Server2的投票为(2, 0)，首先会比较两者的ZXID，均为0，再比较myid，此时Server2的myid最大，于是更新自己的投票为(2, 0)，然后重新投票，对于Server2而言，其无须更新自己的投票，只是再次向集群中所有机器发出上一次投票信息即可。
	
	　　(4) 统计投票。每次投票后，服务器都会统计投票信息，判断是否已经有过半机器接受到相同的投票信息，对于Server1、Server2而言，都统计出集群中已经有两台机器接受了(2, 0)的投票信息，此时便认为已经选出了Leader。
	
	　　(5) 改变服务器状态。一旦确定了Leader，每个服务器就会更新自己的状态，如果是Follower，那么就变更为FOLLOWING，如果是Leader，就变更为LEADING。





**II、服务器运行时期的Leader选举**

　在Zookeeper运行期间，Leader与非Leader服务器各司其职，即便当有非Leader服务器宕机或新加入，此时也不会影响Leader，但是一旦Leader服务器挂了，那么整个集群将暂停对外服务，进入新一轮Leader选举，其过程和启动时期的Leader选举过程基本一致。假设正在运行的有Server1、Server2、Server3三台服务器，当前Leader是Server2，若某一时刻Leader挂了，此时便开始Leader选举。选举过程如下:

	　　(1) 变更状态。Leader挂后，余下的非Observer服务器都会将自己的服务器状态变更为LOOKING，然后开始进入Leader选举过程。
	
	　　(2) 每个Server会发出一个投票。在运行期间，每个服务器上的ZXID可能不同，此时假定Server1的ZXID为123，Server3的ZXID为122；在第一轮投票中，Server1和Server3都会投自己，产生投票(1, 123)，(3, 122)，然后各自将投票发送给集群中所有机器。
	
	　　(3) 接收来自各个服务器的投票。与启动时过程相同。
	
	　　(4) 处理投票。与启动时过程相同，此时，Server1将会成为Leader。
	
	　　(5) 统计投票。与启动时过程相同。
	
	　　(6) 改变服务器的状态。与启动时过程相同。



#### 2.Leader选举算法分析 ####

在Zookeeper中，提供了三种Leader选举的算法，分别是LeaderElection、UDP版本的FastLeaderElection和TCP版本的FastLeaderElection，可以通过在配置文件zoo.cfg中使用electionAlg属性来指定，分别使用数字0～3表示。0代表LeaderElection，这是一种纯UDP实现的Leader选举算法；1代表UDP版本的FastLeaderElection，并且是非授权模式；2也代表UDP版本的FastLeaderElection，但使用授权模式；3代表TCP版本的FastLeaderElection。值得一提的是，从3.4.0版本开始，Zookeeper废弃了0、1和2这三种Leader选举算法，只保留了TCP版本的FastLeaderElection选举算法。下面仅对此算法进行介绍。

首先我们对Zookeeper的Leader选举算法介绍中会出现的一些专有术语进行简单介绍，以便更好地理解Zookeeper的Leader选举算法。

- SID：服务器ID —— SID是一个数字，用来唯一标识一台Zookeeper集群中的机器，每台机器不能重复，和myid的值一致。
- ZXID:事务ID —— ZXID是一个事务ID，用来唯一标识一次服务器状态的变更。
- Vote：投票 —— Leader选举，顾名思义必须通过投票来实现。当集群中的机器发现自己无法检测到Leader机器的时候，就开始尝试进行投票。
- Quorum：过半机器数 —— 指的是Zookeeper集群中过半的机器数，如果集群中总的机器数是n的话，那么 quorum = (n/2 + 1)

#### 1.算法粗略分析 ####

**进入Leader选举**

当Zookeeper集群中的一台服务器出现以下两种情况之一时，就会开始进入Leader选举。

- 服务器初始化启动
- 服务器运行期间无法和Leader保持连接

而当一台机器进入Leader选举流程时，当前集群也可能会处于以下两种状态。

- 集群中本来就已经存在一个Leader
- 集群中确实不存在Leader

第一种已经存在Leader的情况一般都是集群中的某一台机器启动得比较晚，在它启动之前，集群已经可以正常工作，即已经存在了一台Leader服务器。针对这种情况，当该机器试图去选举Leader时，会被告知当前服务器的Leader信息，对于该机器而言，仅仅需要和Leader机器建立起连接，并进行状态同步即可。而第二种在集群中不存在Leader情况下则会相对复杂，下面我们来看一下具体是如何进行Leader选举。

（1）**开始第一次投票**。通常有两种情况会导致集群中不存在Leader，一种情况是在整个服务器刚刚初始化启动时，此时尚未产生一台Leader服务器；另一种情况就是在运行期间当前Leader所在的服务器挂了。无论哪种情况导致进行Leader选举，集群的所有机器都处于一种试图选举出一个Leader的状态，即LOOKING状态，LOOKING状态的机器会向集群中所有其他机器发送消息，这个消息称为“投票”。投票消息中包含了两个最基本的信息：所推举的服务器的SID和ZXID，分别表示了被推举服务器的唯一标识和事务ID。下面我们将以“(SID, ZXID)”这样的形式来标识一次投票信息。举例来说，如果当前服务器要推举SID为1、ZXID为8的服务器成为Leader，那么它的这次投票信息可以表示为（1，8）。我们假定Zookeeper由5台机器组成，SID分别为1、2、3、4、5，ZXID分别为9、9、9、8、8，并且此时SID为2的机器是Leader服务器，某一时刻，1、2所在的机器出现故障，因此集群开始进行Leader选举。在第一次投票的时侯，由于还无法检测到集群中其他机器的状态信息，因此每台机器都会将自己作为被推举的对象来进行投票，于是SID为3、4、5的机器投票情况分别为(3, 9)，(4, 8)， (5, 8)。

（2）**变更投票**。集群中的每台机器发出自己的投票后，也会接收到来自集群中其他机器的投票，每台机器都会根据一定规则来处理收到的其他机器的投票，并以此来决定是否需要变更自己的投票，这个规则也是整个Leader选举算法的核心所在。为了便于描述，我们首先定义一些术语：


-  vote_sid：接收到的投票中所推举Leader服务器的SID。
-  vote_zxid：接收到的投票中所推举Leader服务器的ZXID。
-  self_sid：当前服务器自己的SID。
-  self_zxid：当前服务器自己的ZXID。

每次对于收到的投票的处理，都是一个对（vote_sid,vote_zxid）和（self_sid,self_zxid）对比的过程。

		

- 规则一：如果vote_zxid大于self_zxid，就认可当前收到的投票，并再次将该投票发送出去。
- 规则二：如果vote_zxid小于self_zxid，那么坚持自己的投票，不做任何变更。
- 规则三：如果vote_zxid等于self_zxid，那么就对比两者的SID，如果vote_sid大于self_sid，那么就认可当前收到的投票，并再次将该投票发送出去。
- 规则四：如果vote_zxid等于self_zxid，并且vote_sid小于self_sid，那么坚持自己的投票，不做任何变更。

根据上面这个规则，给出下面Zookeeper集群的投票变更过程。

![](http://images2015.cnblogs.com/blog/616953/201612/616953-20161202213100568-693960760.png)

（3）**确定Leader**。经过第二轮投票后，集群中的每台机器都会再次接收到其他机器的投票，然后开始统计投票，如果一台机器收到了超过半数的相同投票，那么这个投票对应的SID机器即为Leader。此时Server3将成为Leader。


简单地说，通常哪台服务器上的数据越新，那么越有可能成为Leader，原因很简单，数据越新，那么它的ZXID也就越大，也就越能够保证数据的恢复。当然，如果集群中有几个服务器具有相同的哦ZXID，那么SID较大的那台服务器成为Leader。 （注意：前提是同一轮投票）

#### 2.Leader选举的实现细节 ####

**I、服务器状态**

为了能够清楚地对Zookeeper集群中每台机器的状态进行标识，在org.apache.zooKeeper.server.quorum.QuorumPeer.ServerState类中列举了4中服务器状态，分别是：LOOKING、FOLLOWING、LEADING和OBSERVING。

- LOOKING：寻找Leader状态。当服务器处于该状态时，它会认为当前集群中没有Leader，因此需要进入Leader选举状态。
- FOLLOWING：跟随者状态。表明当前服务器角色是Follower。
- LEADING：领导者状态。表明当前服务器角色是Leader。
- OBSERVING：观察者状态。表明当前服务器角色是Observer。

**II、投票数据结构**

![](https://i.imgur.com/B6Ird0v.png)


- id：被推举的Leader的SID。
- zxid：被推举的Leader事务ID。
- electionEpoch：逻辑时钟，用来判断多个投票是否在同一轮选举周期中，该值在服务端是一个自增序列，每次进入新一轮的投票后，都会对该值进行加1操作。
- peerEpoch：被推举的Leader的epoch。
- state：当前服务器的状态。

**III、QuorumCnxManager:网络IO**

每台服务器在启动的过程中，会启动一个QuorumPeerManager，负责各台服务器之间的底层Leader选举过程中的网络通信。

（1）**消息队列**。QuorumCnxManager内部维护了一系列的队列，用来保存接收到的、待发送的消息以及消息的发送器，除接收队列以外，其他队列都按照SID分组形成队列集合，如一个集群中除了自身还有3台机器，那么就会为这3台机器分别创建一个发送队列，互不干扰。



- recvQueue：消息接收队列，用于存放那些从其他服务器接收到的消息。
- queueSendMap：消息发送队列，用于保存那些待发送的消息，按照SID进行分组。
- senderWorkerMap：发送器集合，每个SenderWorker消息发送器，都对应一台远程Zookeeper服务器，负责消息的发送，也按照SID进行分组。
- lastMessageSent：最近发送过的消息，为每个SID保留最近发送过的一个消息。


（2）**建立连接**。为了能够进行相互投票，Zookeeper集群中的所有机器都需要两两建立起网络连接。QuorumCnxManager在启动时会创建一个ServerSocket来监听Leader选举的通信端口(默认为3888)。开启端口监听后，Zookeeper能够不断地接收到来自其他服务器的创建连接请求，在接收到其他服务器的TCP连接请求时，会交由receiveConnection函数来处理。为了避免两台机器之间重复地创建TCP连接，Zookeeper只允许SID大的服务器主动和其他机器建立连接，否则断开连接。在接收到创建连接请求后，服务器通过对比自己和远程服务器的SID值来判断是否接收连接请求，如果当前服务器发现自己的SID更大，那么会断开当前连接，然后自己主动和远程服务器建立连接。一旦连接建立，就会根据远程服务器的SID来创建相应的消息发送器SendWorker和消息接收器RecvWorker，并启动他们。

(3) **消息接收与发送**。消息接收过程是由消息接收器RecvWorker来负责的，由于Zookeeper会为每个远程服务器都分配一个单独的RecvWorker，因此，每个RecvWorker只需要不断地从这个TCP连接中读取消息，并将其保存到recvQueue队列中。消息发送过程也比较简单，由于Zookeeper同样也已经为每个远程服务器都分配一个单独的SendWorker，因此，每个SendWorker只需要不断地从对应的消息发送队列中获取出一个消息发送即可，同时将这个消息放入lastMessageSent中来作为最近发送过的消息。在SendWorker的具体实现中，一旦Zookeeper发现针对当前服务器的消息发送队列为空，那么此时需要从lastMessageSent中取出一个最近发送过的消息来进行再次发送，这是为了解决接收方在消息接收前或者接收到消息后服务器挂了，导致消息尚未被正确处理。那么如此重复发送是否会导致其他问题呢？这里可以放心的一点是，Zookeeper能够保证接收方在处理消息时，会对重复消息进行正确的处理。


**IV、FastLeaderElection：选举算法的核心部分**

我们首先约定几个概念：

- 外部投票：特指其他服务器发来的投票。
-  内部投票：服务器自身当前的投票。
-   选举轮次：Zookeeper服务器Leader选举的轮次，即logicalclock。
-   PK：指对内部投票和外部投票进行对比来确定是否需要变更内部投票。

**I、选票管理**

- sendqueue：选票发送队列，用于保存待发送的选票。
- recvqueue：选票接收队列，用于保存接收到的外部投票。
- WorkerReceiver：选票接收器。其会不断地从QuorumCnxManager中获取其他服务器发来的选举消息，并将其转换成一个选票，然后保存到recvqueue中，在选票接收过程中，如果发现该外部选票的选举轮次小于当前服务器的，那么忽略该外部投票，同时立即发送自己的内部投票。
-  WorkerSender：选票发送器，会不断地从sendqueue队列中获取待发送的选票，并将其传递到底层QuorumCnxManager中去。

**II、Leader选举算法核心**

![](http://images2015.cnblogs.com/blog/616953/201612/616953-20161206114702772-1120304539.png)

从上图中，我们可以看到FastLeaderElction模块是如何与底层的网络IO进行交互的。下面我们来看下这个选举过程的核心算法实现的流程：

![](https://i.imgur.com/JOcERag.png)

上图中展示了Leader选举算法的基本流程，其实也就是lookForLeader方法的逻辑。当Zookeeper服务器检测到当前服务器状态变成LOOKING时，就会触发Leader选举，即调用lookForLeader方法来进行Leader选举。

	

- 　1）**自增选举轮次**。在FastLeaderElection实现中，有一个logicalclock属性，用于标识当前Leader的选举轮次，Zookeeper规定了所有有效的投票都必须在同一轮次中，Zookeeper在开始新一轮投票时，会首先对logicalclock进行自增操作。

-  2）**初始化选票**。在开始进行新一轮投票之前，每个服务器都会首先初始化自己的选票，并且在初始化阶段，每台服务器都会将自己推举为Leader。
	
- 3） **发送初始化选票**。完成选票的初始化后，服务器就会发起第一次投票。Zookeeper会将刚刚初始化好的选票放入sendqueue中，由发送器WorkerSender负责发送出去。
	
- 4）**接收外部投票**。每台服务器会不断地从recvqueue队列中获取外部选票。如果服务器发现无法获取到任何外部投票，那么就会立即确认自己是否和集群中其他服务器保持着有效的连接，如果没有连接，则马上建立连接，如果已经建立了连接，则再次发送自己当前的内部投票。
	


- 5）**判断选举轮次**。在发送完初始化选票之后，接着开始处理外部投票。在处理外部投票时，会根据选举轮次来进行不同的处理。
	
		　　　　a. 外部投票的选举轮次大于内部投票。 —— 若服务器自身的选举轮次落后于该外部投票对应服务器的选举轮次，那么就会立即更新自己的选举轮次(logicalclock)，并且清空所有已经收到的投票，然后使用初始化的投票来进行PK以确定是否变更内部投票。最终再将内部投票发送出去。
		
		　　　　b. 外部投票的选举轮次小于内部投票。若服务器接收的外选票的选举轮次落后于自身的选举轮次，那么Zookeeper就会直接忽略该外部投票，不做任何处理，并返回步骤4。
		
		　　　　c. 外部投票的选举轮次等于内部投票。此时可以开始进行选票PK。
	


- 6）**选票PK**。在进行选票PK时，符合任意一个条件就需要变更投票。
	
		　　　　a. 若外部投票中推举的Leader服务器的选举轮次大于内部投票，那么需要变更投票。
		
		　　　　b. 若选举轮次一致，那么就对比两者的ZXID，若外部投票的ZXID大，那么需要变更投票。
		
		　　　　c. 若两者的ZXID一致，那么就对比两者的SID，若外部投票的SID大，那么就需要变更投票。
	


- 7) **变更投票**。经过PK后，若确定了外部投票优于内部投票，那么就变更投票，即使用外部投票的选票信息来覆盖内部投票，变更完成后，再次将这个变更后的内部投票发送出去。
	


- 8) **选票归档**。无论是否变更了投票，都会将刚刚收到的那份外部投票放入选票集合recvset中进行归档。recvset用于记录当前服务器在本轮次的Leader选举中收到的所有外部投票（按照服务队的SID区别，如{(1, vote1), (2, vote2)...}）。
	


- 9) **统计投票**。完成选票归档后，就可以开始统计投票，统计投票是为了统计集群中是否已经有过半的服务器认可了当前的内部投票，如果确定已经有过半服务器认可了该投票，则终止投票。否则返回步骤4。
	


- 10) **更新服务器状态**。若已经确定可以终止投票，那么就开始更新服务器状态，服务器首选判断当前被过半服务器认可的投票所对应的Leader服务器是否是自己，若是自己，则将自己的服务器状态更新为LEADING，若不是，则根据具体情况来确定自己是FOLLOWING或是OBSERVING。

以上10个步骤，就是FastLeaderElection选举算法的核心步骤，其中步骤4～9会经过几轮循环，直到Leader选举产生。（另外还有一个细节需要注意，就是在完成步骤9之后，如果统计投票发现已经有过半的服务器认可了当前的选票，这个时候，Zookeeper并不会立即进入步骤10来更新服务器状态，而是会等待一段时间（默认是200毫秒）来确定是否有新的更优的投票）。






### 七、Zookeeper各服务器角色介绍 ###

#### 1.Leader ####

Leader服务器是整个Zookeeper集群工作机制中的核心，其主要工作有以下两个：

- 事务请求的唯一调度和处理者，保证集群事务处理的顺序性。
- 集群内部各服务器的调度者。

**I、请求处理链**

使用责任链模式来处理每一个客户端请求是Zookeeper的一大特色。在每一个服务器启动的时候，都会进行请求处理链的初始化，Leader服务器的请求处理链如下：

![](http://images2015.cnblogs.com/blog/616953/201612/616953-20161206203032319-1806823400.png)



- 　 **PrepRequestProcessor**。PrepRequestProcessor是Leader服务器的请求预处理器，也是Leader服务器的第一个请求处理器。在Zookeeper中，我们将那些会改变服务器状态的请求称为事务请求（创建节点、更新数据、删除节点、创建会话等），PrepRequestProcessor能够识别出当前客户端请求是否是事务请求。对于事务请求，PrepRequestProcessor处理器会对其进行一系列预处理，诸如创建请求事务头、事务体、会话检查、ACL检查和版本检查等。

- 　 **ProposalRequestProcessor**。ProposalRequestProcessor处理器是Leader服务器的事务投票处理器，也是Leader服务器事务处理流程的发起者。对于非事务性请求，ProposalRequestProcessor会直接将请求转发到CommitProcessor处理器，不再做任何处理；而对于事务性请求，除了将请求转发到CommitProcessor外，还会根据请求类型创建对应的Proposal提议，并发送给所有的Follower服务器来发起一次集群内的事务投票。同时，ProposalRequestProcessor还会将事务请求交付给SyncRequestProcessor进行事务日志的记录。

-  **SyncRequestProcessor**。SyncRequestProcessor是事务日志记录处理器。该处理器用来将事务请求记录到事务日志文件中，同时还会触发Zookeeper进行数据快照。

-  **AckRequestProcessor**。AckRequestProcessor处理器是Leader特有的处理器，其主要负责在SyncRequestProcessor完成事务日志记录后，向Proposal的投票收集器发送ACK反馈，以通知投票收集器当前服务器已经完成了对该Proposal的事务日志记录。

-  **CommitProcessor**。CommitProcessor是事务提交处理器。对于非事务请求，该处理器会直接将其交付给下一级处理器处理；而对于事务请求，其会等待集群内针对Proposal的投票直到该Proposal可被提交，利用CommitProcessor处理器，每个服务器都可以很好地控制对事务请求的顺序处理。

-  **ToBeCommitProcessor**。ToBeCommitProcessor处理器有一个toBeApplied队列，专门用来存储那些已经被CommitProcessor处理过的可被提交的Proposal。ToBeCommitProcessor处理器会将这些请求逐个交付给FinalRequestProcessor处理器进行处理，待其处理完后，再将其从toBeApplied队列中移除。

-  **FinalRequestProcessor**。FinalRequestProcessor是最后一个请求处理器。该处理器主要用来进行客户端请求返回之前的收尾操作，包括创建客户端请求的响应；针对事务请求，该处理还会负责将事务应用到内存数据库中去。


**II、LearnerHandler**

为了保证整个集群内部的实时通信，同时为了确保可以控制所有的Follower/Observer服务器，Leader服务器会与每个Follower/Observer服务器建立一个TCP长连接。同时也会为每个Follower/Observer服务器创建一个名为LearnerHandler的实体。

LearnerHandler，顾名思义，是Zookeeper集群中Learner服务器的管理者，主要负责Follower/Observer服务器和Leader服务器之间的一系列网络通信，包括数据同步、请求转发和Proposal提议的投票等。Leader服务器中保存了所有Follower/Observer对应的LearnerHandler。


#### 2.Follower ####

Follower服务器是Zookeeper集群状态的跟随着，其主要工作有以下三个：

- 处理客户端非事务性请求（读取数据），转发事务请求给Leader服务器。
- 参与事务请求Proposal的投票。
- 参与Leader选举投票。

和Leader服务器一样，Follower也采用了责任链模式组装的请求处理链来处理每一个客户端请求，由于不需要对事务请求的投票处理，因此Follower的请求处理链会相对简单，其处理链如下：

![](http://images2015.cnblogs.com/blog/616953/201612/616953-20161206205916319-94850171.png)

-  **FollowerRequestProcessor**。FollowerRequestProcessor是Follower服务器的第一请求处理器，其主要工作就是识别当前请求是否是事务请求，若是，那么Follower就会将该请求转发给Leader服务器，Leader服务器在接收到这个事务请求后，就会将其提交到请求处理链，按照正常事务请求进行处理。

- **SendAckRequestProcessor**。SendAckRequestProcessor是Follower服务器上另外一个和Leader服务器有差异的请求处理器。我们讲到过Leader服务器上有一个叫AckRequestProcessor的请求处理器，其主要负责在SyncRequestProcessor处理器完成事务日志记录后，向Proposal的投票收集器进行反馈。而在Follower服务器上，SendAckRequestProcessor处理器同样承担了事务日志记录反馈的角色，在完成事务日志记录后，会向Leader服务器发送ACK消息以表明自身完成了事务日志的记录工作。两者的唯一区别在于，AckRequestProcessor处理器和Leader服务器在同一个服务器上，因此它的ACK反馈仅仅是一个本地操作；而SendAckRequestProcessor处理器由于在Follower服务器上，因此需要通过以ACK消息的形式来向Leader服务器进行反馈。  


#### 3.Observer ####

Observer充当观察者角色，观察Zookeeper集群的最新状态变化并将这些状态同步过来。Observer服务器在工作原理上和Follower基本是一致的，对于非事务请求都可以进行独立处理，而对于事务请求，则会转发给Leader服务器进行处理。Observer不会参与任何形式的投票，包括事务请求Proposal的投票和Leader选举投票。简单地讲，Observer服务器只提供非事务服务，通常用于在不影响集群事务处理能力的前提下提升集群的非事务处理能力。


另外，Observer的请求处理链路和Follower服务器也非常相近。其处理链如下（需要注意一点是，虽然在图中，Observer服务器在初始化阶段会将SynRequestProcessor处理器也组装上去，但是在实际运行过程中，Leader服务器不会将事务请求的投票发送给Observer服务器）：

![](https://i.imgur.com/Dm60pzV.png)



### 八、Zookeeper请求处理 ###

### （一）、会话创建请求 ###

![](http://images2015.cnblogs.com/blog/616953/201612/616953-20161207095042507-929689583.png)

#### 1. 请求接收 ####

　　(1) I/O层接收来自客户端的请求。NIOServerCnxn维护每一个客户端连接，客户端与服务器端的所有通信都是由NIOServerCnxn负责，其负责统一接收来自客户端的所有请求，并将请求内容从底层网络I/O中完整地读取出来。

　　(2) 判断是否是客户端会话创建请求。每个会话对应一个NIOServerCnxn实体，对于每个请求，Zookeeper都会检查当前NIOServerCnxn实体是否已经被初始化，如果尚未被初始化，那么就可以确定该客户端一定是会话创建请求。

　　(3) 反序列化ConnectRequest请求。一旦确定客户端请求是否是会话创建请求，那么服务端就可以对其进行反序列化，并生成一个ConnectRequest载体。

　　(4) 判断是否是ReadOnly客户端。如果当前Zookeeper服务器是以ReadOnly模式启动，那么所有来自非ReadOnly型客户端的请求将无法被处理。因此，服务端需要先检查是否是ReadOnly客户端，并以此来决定是否接受该会话创建请求。

　　(5) 检查客户端ZXID。正常情况下，在一个Zookeeper集群中，服务端的ZXID必定大于客户端的ZXID，因此若发现客户端的ZXID大于服务端ZXID，那么服务端不接受该客户端的会话创建请求。

　　(6) 协商sessionTimeout。在客户端向服务器发送超时时间后，服务器会根据自己的超时时间限制最终确定该会话超时时间，这个过程就是sessionTimeout协商过程。

　　(7) 判断是否需要重新激活创建会话。服务端根据客户端请求中是否包含sessionID来判断该客户端是否需要重新创建会话，若客户单请求中包含sessionID，那么就认为该客户端正在进行会话重连，这种情况下，服务端只需要重新打开这个会话，否则需要重新创建。

#### 2. 会话创建 ####

　　(1) 为客户端生成sessionID。在为客户端创建会话之前，服务端首先会为每个客户端分配一个sessionID，服务端为客户端分配的sessionID是全局唯一的。

　　(2) 注册会话。向SessionTracker中注册会话，SessionTracker中维护了sessionsWithTimeout和sessionsById，在会话创建初期，会将客户端会话的相关信息保存到这两个数据结构中。

　　(3) 激活会话。激活会话涉及Zookeeper会话管理的分桶策略，其核心是为会话安排一个区块，以便会话清理程序能够快速高效地进行会话清理。

　　(4) 生成会话密码。服务端在创建一个客户端会话时，会同时为客户端生成一个会话密码，连同sessionID一同发给客户端，作为会话在集群中不同机器间转移的凭证。

#### 3. 预处理 ####

　　(1) 将请求交给PrepRequestProcessor处理器处理。在提交给第一个请求处理器之前，Zookeeper会根据该请求所属的会话，进行一次激活会话操作，以确保当前会话处于激活状态，完成会话激活后，则提交请求至处理器。

　　(2) 创建请求事务头。对于事务请求，Zookeeper会为其创建请求事务头，服务端后续的请求处理器都是基于该请求头来识别当前请求是否是事务请求，请求事务头包含了一个事务请求最基本的一些信息，包括sessionID、ZXID（事务请求对应的事务ZXID）、CXID（客户端的操作序列）和请求类型（如create、delete、setData、createSession等）等。

　　(3) 创建请求事务体。由于此时是会话创建请求，其事务体是CreateSessionTxn。

　　(4) 注册和激活会话。处理由非Leader服务器转发过来的会话创建请求。

#### 4. 事务处理 ####

　　(1) 将请求交给ProposalRequestProcessor处理器。与提议相关的处理器，从ProposalRequestProcessor开始，请求的处理将会进入三个子处理流程，分别是Sync流程、Proposal流程、Commit流程。

![](http://images2015.cnblogs.com/blog/616953/201612/616953-20161207103040257-77514359.png)


#### 5. 事务应用 ####

　　(1) 交付给FinalRequestProcessor处理器。FinalRequestProcessor处理器检查outstandingChanges队列中请求的有效性，若发现这些请求已经落后于当前正在处理的请求，那么直接从outstandingChanges队列中移除。

　　(2) 事务应用。之前的请求处理仅仅将事务请求记录到了事务日志中，而内存数据库中的状态尚未改变，因此，需要将事务变更应用到内存数据库。

　　(3) 将事务请求放入队列commitProposal。完成事务应用后，则将该请求放入commitProposal队列中，commitProposal用来保存最近被提交的事务请求，以便集群间机器进行数据的快速同步。

　　#### 6. 会话响应 ####

　　(1) 统计处理。Zookeeper计算请求在服务端处理所花费的时间，统计客户端连接的基本信息，如lastZxid(最新的ZXID)、lastOp(最后一次和服务端的操作)、lastLatency(最后一次请求处理所花费的时间)等。

　　(2) 创建响应ConnectResponse。会话创建成功后的响应，包含了当前客户端和服务端之间的通信协议版本号、会话超时时间、sessionID和会话密码。

　　(3) 序列化ConnectResponse。

　　(4) I/O层发送响应给客户端。


### （二）、SetData请求 ###

![](https://i.imgur.com/DHN5oxC.png)

服务端对于SetData请求大致可以分为四步，预处理、事务处理、事务应用、请求响应。

#### 　　1. 预处理 ####

　　(1) I/O层接收来自客户端的请求。

　　(2) 判断是否是客户端"会话创建"请求。对于SetData请求，按照正常事务请求进行处理。

　　(3) 将请求交给PrepRequestProcessor处理器进行处理。

　　(4) 创建请求事务头。

　　(5) 会话检查。检查该会话是否有效。

　　(6) 反序列化请求，并创建ChangeRecord记录。反序列化并生成特定的SetDataRequest请求，请求中包含了数据节点路径path、更新的内容data和期望的数据节点版本version。同时根据请求对应的path，Zookeeper生成一个ChangeRecord记录，并放入outstandingChanges队列中。

　　(7) ACL检查。检查客户端是否具有数据更新的权限。

　　(8) 数据版本检查。通过version属性来实现乐观锁机制的写入校验。

　　(9) 创建请求事务体SetDataTxn。

　　(10) 保存事务操作到outstandingChanges队列中。

#### 　　2. 事务处理 ####

　　对于事务请求，服务端都会发起事务处理流程。所有事务请求都是由ProposalRequestProcessor处理器处理，通过Sync、Proposal、Commit三个子流程相互协作完成。

#### 　　3. 事务应用 ####

　　(1) 交付给FinalRequestProcessor处理器。

　　(2) 事务应用。将请求事务头和事务体直接交给内存数据库ZKDatabase进行事务应用，同时返回ProcessTxnResult对象，包含了数据节点内容更新后的stat。

　　(3) 将事务请求放入commitProposal队列。

#### 　　4. 请求响应 ####

　　(1) 创建响应体SetDataResponse。其包含了当前数据节点的最新状态stat。

　　(2) 创建响应头。包含当前响应对应的事务ZXID和请求处理是否成功的标识。

　　(3) 序列化响应。

　　(4) I/O层发送响应给客户端。


### （三）、GetData请求 ###

![](https://i.imgur.com/9R2FsX7.png)

服务端对于GetData请求的处理，大致分为三步，预处理、非事务处理、请求响应。

#### 　　1. 预处理 ####

　　(1) I/O层接收来自客户端的请求。

　　(2) 判断是否是客户端"会话创建"请求。

　　(3) 将请求交给PrepRequestProcessor处理器进行处理。

　　(4) 会话检查。

#### 　　2. 非事务处理 ####

　　(1) 反序列化GetDataRequest请求。

　　(2) 获取数据节点。

　　(3) ACL检查。

　　(4) 获取数据内容和stat，注册Watcher。

#### 　　3. 请求响应 ####

　　(1) 创建响应体GetDataResponse。响应体包含当前数据节点的内容和状态stat。

　　(2) 创建响应头。

　　(3) 统计处理。

　　(4) 序列化响应。

　　(5) I/O层发送响应给客户端。


### 九、Zookeeper数据与存储 ###

最后，让我们来看下Zookeeper最底层数据与存储的技术内幕。在Zookeeper中，数据存储分为两部分：内存数据存储于磁盘数据存储。

### （一）、内存数据 ###

![](https://i.imgur.com/QDFRJNX.png)

#### I、DataTree ####

DataTree是Zookeeper内存数据存储的核心，是一个树的数据结构，代表了内存中的一份完整的数据。DataTree不包含任何与网络、客户端连接及请求处理等相关的业务逻辑，是一个非常独立的Zookeeper组件。

DataTree底层的数据结构其实是一个典型的ConcurrentHashMap键值对结构：

	private final ConcurrentHashMap<String,DataNode> nodes = new ConcurrentHashMap<String,DataNode>();

在nodes这个Map中，存放了Zookeeper服务器上所有的数据节点，可以说，对于Zookeeper数据的所有 操作，底层都是对这个Map结构的操作。nodes以数据节点的路径（path）为key,value则是节点的数据内容：DataNode。

另外，对于所有的临时节点，为了方便实时访问和及时清理，DataTree中还单独将临时节点保存起来：

	private final Map<Long,HashSet<String>> ephemerals = new ConcurrentHashMap<Long,HashSet<String>>()

#### II、DataNode ####

DataNode是数据存储的最小单元，其内部除了保存了结点的数据内容、ACL列表、节点状态之外，还记录了父节点的引用和子节点列表两个属性，同时，DataNode还提供了对子节点列表进行操作的各个接口。

#### II、ZKDatabase ####

Zookeeper的内存数据库，管理Zookeeper的所有会话、DataTree存储和事务日志。ZKDatabase会定时向磁盘dump快照数据，同时在Zookeeper启动时，会通过磁盘的事务日志和快照文件恢复成一个完整的内存数据库。


### （二）、事务日志 ###

我们已经多次提到了Zookeeper的事务日志。下面我们将从事务日志的存储、日志格式和日志写入过程几个方面来深入讲解Zookeeper底层实现数据一致性过程中最重要的一部分。

#### I、文件存储 ####

在配置Zookeeper集群时需要配置dataDir目录，其用来存储事务日志文件。也可以为事务日志单独分配一个文件存储目录:dataLogDir。若配置dataLogDir为/home/admin/zkData/zk_log，那么Zookeeper在运行过程中会在该目录下建立一个名字为version-2的子目录，该目录确定了当前Zookeeper使用的事务日志格式版本号，当下次某个Zookeeper版本对事务日志格式进行变更时，此目录也会变更，即在version-2子目录下会生成一系列文件大小一致(64MB)的文件。

#### II、日志格式 ####

下面我们来看看这个事务日志里面到底有些什么内容。为此，我们首先部署一个全新的Zookeeper服务器，配置相关的事务日志存储目录，启动之后，进行一些事务操作。如：

	(1) 创建/test_log节点，初始值为v1。
	
	(2) 更新/test_log节点的数据为v2。
	
	(3) 创建/test_log/c节点，初始值为v1。
	
	(4) 删除/test_log/c节点。


经过如上事务操作后，在Zookeeper事务日志存储目录中就可以看到产生了一个事务日志，使用二进制编辑器将这个文件打开后，就可以看到如下序列化之后的事务日志了。

![](https://i.imgur.com/zf8a1Du.png)

对于这个事务日志，我们无法直接通过肉眼识别出其究竟包含了哪些事务操作，但可以通过某种方式将这些事务日志转换成正常日志文件，以便让开发与运维人员能够清楚地看明白Zookeeper的事务操作。Zookeeper提供了一套简易的事务日志格式化工具org.apache.zookeeper.Server.LogFormatter,用于将这个默认的事务日志文件转换成可视化的事务操作日志 —— （将Zookeeper下的zookeeper-3.4..jar 和 slf4j-api-1.6.1.jar 复制到 .../version-2目录下，使用如下命令打开：java -classpath ./zookeeper-3.4.6.jar:./slf4j-api-1.6.1.jar org.apache.zookeeper.server.LogFormatter log.300000001），执行后的输出结果如下所示：

![](https://i.imgur.com/Z9lMVvg.png)


#### III、日志写入 ####

　FileTxnLog负责维护事务日志对外的接口，包括事务日志的写入和读取等。Zookeeper的事务日志写入过程大体可以分为如下6个步骤。

　　(1) **确定是否有事务日志可写**。当Zookeeper服务器启动完成需要进行第一次事务日志的写入，或是上一次事务日志写满时，都会处于与事务日志文件断开的状态，即Zookeeper服务器没有和任意一个日志文件相关联。因此在进行事务日志写入前，Zookeeper首先会判断FileTxnLog组件是否已经关联上一个可写的事务日志文件。若没有，则会使用该事务操作关联的ZXID作为后缀创建一个事务日志文件，同时构建事务日志的文件头信息，并立即写入这个事务日志文件中去，同时将该文件的文件流放入streamToFlush集合，该集合用来记录当前需要强制进行数据落盘的文件流。

　　(2) **确定事务日志文件是否需要扩容**(预分配)。Zookeeper会采用磁盘空间预分配策略。当检测到当前事务日志文件剩余空间不足4096字节时，就会开始进行文件空间扩容，即在现有文件大小上，将文件增加65536KB(64MB)，然后使用"0"填充被扩容的文件空间。

　　(3) **事务序列化**。对事务头和事务体的序列化，其中事务体又可分为会话创建事务、节点创建事务、节点删除事务、节点数据更新事务等。

　　(4) **生成Checksum**。为保证日志文件的完整性和数据的准确性，Zookeeper在将事务日志写入文件前，会计算生成Checksum。

　　(5) **写入事务日志文件流**。将序列化后的事务头、事务体和Checksum写入文件流中，此时并为写入到磁盘上。

　　(6) **事务日志刷入磁盘**。由于步骤5中的缓存原因，无法实时地写入磁盘文件中，因此需要将缓存数据强制刷入磁盘。


#### IV、日志截断 ####

在Zookeeper运行过程中，可能出现非Leader记录的事务ID比Leader上大，这是非法运行状态。此时，需要保证所有机器必须与该Leader的数据保持同步，即Leader会发送TRUNC命令给该机器，要求进行日志截断，Learner收到该命令后，就会删除所有包含或大于该事务ID的事务日志文件。


### （三）、snapshot —— 数据快照 ###

数据快照是Zookeeper数据存储中另一个非常核心的运行机制，数据快照用来记录Zookeeper服务器上某一时刻的全量内存数据内容，并将其写入指定的磁盘文件中。

#### 1. 文件存储 ####

　　与事务文件类似，Zookeeper快照文件也可以指定特定磁盘目录，通过dataDir属性来配置。若指定dataDir为/home/admin/zkData/zk_data，则在运行过程中会在该目录下创建version-2的目录，该目录确定了当前Zookeeper使用的快照数据格式版本号。在Zookeeper运行时，会生成一系列文件。

#### 　　2. 数据快照 ####

　　FileSnap负责维护快照数据对外的接口，包括快照数据的写入和读取等，将内存数据库写入快照数据文件其实是一个序列化过程。针对客户端的每一次事务操作，Zookeeper都会将他们记录到事务日志中，同时也会将数据变更应用到内存数据库中，Zookeeper在进行若干次事务日志记录后，将内存数据库的全量数据Dump到本地文件中，这就是数据快照。其步骤如下

　　(1) 确定是否需要进行数据快照。每进行一次事务日志记录之后，Zookeeper都会检测当前是否需要进行数据快照，考虑到数据快照对于Zookeeper机器的影响，需要尽量避免Zookeeper集群中的所有机器在同一时刻进行数据快照。采用过半随机策略进行数据快照操作。

　　(2) 切换事务日志文件。表示当前的事务日志已经写满，需要重新创建一个新的事务日志。

　　(3) 创建数据快照异步线程。创建单独的异步线程来进行数据快照以避免影响Zookeeper主流程。

　　(4) 获取全量数据和会话信息。从ZKDatabase中获取到DataTree和会话信息。

　　(5) 生成快照数据文件名。Zookeeper根据当前已经提交的最大ZXID来生成数据快照文件名。

　　(6) 数据序列化。首先序列化文件头信息，然后再对会话信息和DataTree分别进行序列化，同时生成一个Checksum，一并写入快照数据文件中去。


### （四）、初始化 ###

在Zookeeper服务器启动期间，首先会进行数据初始化工作，用于将存储在磁盘上的数据文件加载到Zookeeper服务器内存中。

![](http://images2015.cnblogs.com/blog/616953/201612/616953-20161216214526120-1735909566.png)

数据的初始化工作是从磁盘上加载数据的过程，主要包括了从快照文件中加载快照数据和根据事务日志进行数据修正两个过程。

　　(1) 初始化FileTxnSnapLog。FileTxnSnapLog是Zookeeper事务日志和快照数据访问层，用于衔接上层业务和底层数据存储，底层数据包含了事务日志和快照数据两部分。FileTxnSnapLog中对应FileTxnLog和FileSnap。

　　(2) 初始化ZKDatabase。首先构建DataTree，同时将FileTxnSnapLog交付ZKDatabase，以便内存数据库能够对事务日志和快照数据进行访问。在ZKDatabase初始化时，DataTree也会进行相应的初始化工作，如创建一些默认结点，如/、/zookeeper、/zookeeper/quota三个节点。

　　(3) 创建PlayBackListener。其主要用来接收事务应用过程中的回调，在Zookeeper数据恢复后期，会有事务修正过程，此过程会回调PlayBackListener来进行对应的数据修正。

　　(4) 处理快照文件。此时可以从磁盘中恢复数据了，首先从快照文件开始加载。

　　(5) 获取最新的100个快照文件。更新时间最晚的快照文件包含了最新的全量数据。

　　(6) 解析快照文件。逐个解析快照文件，此时需要进行反序列化，生成DataTree和sessionsWithTimeouts，同时还会校验Checksum及快照文件的正确性。对于100个快找文件，如果正确性校验通过时，通常只会解析最新的那个快照文件。只有最新快照文件不可用时，才会逐个进行解析，直至100个快照文件全部解析完。若将100个快照文件解析完后还是无法成功恢复一个完整的DataTree和sessionWithTimeouts，此时服务器启动失败。

　　(7) 获取最新的ZXID。此时根据快照文件的文件名即可解析出最新的ZXID：zxid_for_snap。该ZXID代表了Zookeeper开始进行数据快照的时刻。

　　(8) 处理事务日志。此时服务器内存中已经有了一份近似全量的数据，现在开始通过事务日志来更新增量数据。

　　(9) 获取所有zxid_for_snap之后提交的事务。此时，已经可以获取快照数据的最新ZXID。只需要从事务日志中获取所有ZXID比步骤7得到的ZXID大的事务操作。

　　(10) 事务应用。获取大于zxid_for_snap的事务后，将其逐个应用到之前基于快照数据文件恢复出来的DataTree和sessionsWithTimeouts。每当有一个事务被应用到内存数据库中后，Zookeeper同时会回调PlayBackListener，将这事务操作记录转换成Proposal，并保存到ZKDatabase的committedLog中，以便Follower进行快速同步。

　　(11) 获取最新的ZXID。待所有的事务都被完整地应用到内存数据库中后，也就基本上完成了数据的初始化过程，此时再次获取ZXID，用来标识上次服务器正常运行时提交的最大事务ID。

　　(12) 校验epoch。epoch标识了当前Leader周期，集群机器相互通信时，会带上这个epoch以确保彼此在同一个Leader周期中。完成数据加载后，Zookeeper会从步骤11中确定ZXID中解析出事务处理的Leader周期：epochOfZxid。同时也会从磁盘的currentEpoch和acceptedEpoch文件中读取上次记录的最新的epoch值，进行校验。



### （五）、数据同步 ###

![](http://images2015.cnblogs.com/blog/616953/201612/616953-20161217172721136-1675752544.png)

　(1) 获取Learner状态。在注册Learner的最后阶段，Learner服务器会发送给Leader服务器一个ACKEPOCH数据包，Leader会从这个数据包中解析出该Learner的currentEpoch和lastZxid。

　(2) 数据同步初始化。首先从Zookeeper内存数据库中提取出事务请求对应的提议缓存队列proposals，同时完成peerLastZxid(该Learner最后处理的ZXID)、minCommittedLog(Leader提议缓存队列commitedLog中最小的ZXID)、maxCommittedLog(Leader提议缓存队列commitedLog中的最大ZXID)三个ZXID值的初始化。

对于集群数据同步而言，通常分为四类，直接差异化同步(DIFF同步)、先回滚再差异化同步(TRUNC+DIFF同步)、仅回滚同步(TRUNC同步)、全量同步(SNAP同步)。



- 直接差异化同步(DIFF同步，peerLastZxid介于minCommittedLog和maxCommittedLog之间)。Leader首先向这个Learner发送一个DIFF指令，用于通知Learner进入差异化数据同步阶段，Leader即将把一些Proposal同步给自己，针对每个Proposal，Leader都会通过发送PROPOSAL内容数据包和COMMIT指令数据包来完成，



- 先回滚再差异化同步(TRUNC+DIFF同步，Leader已经将事务记录到本地事务日志中，但是没有成功发起Proposal流程)。当Leader发现某个Learner包含了一条自己没有的事务记录，那么就需要该Learner进行事务回滚，回滚到Leader服务器上存在的，同时也是最接近于peerLastZxid的ZXID。



-  仅回滚同步(TRUNC同步，peerLastZxid大于maxCommittedLog)。Leader要求Learner回滚到ZXID值为maxCommittedLog对应的事务操作。



- 全量同步(SNAP同步，peerLastZxid小于minCommittedLog或peerLastZxid不等于lastProcessedZxid)。Leader无法直接使用提议缓存队列和Learner进行同步，因此只能进行全量同步。Leader将本机的全量内存数据同步给Learner。Leader首先向Learner发送一个SNAP指令，通知Learner即将进行全量同步，随后，Leader会从内存数据库中获取到全量的数据节点和会话超时时间记录器，将他们序列化后传输给Learner。Learner接收到该全量数据后，会对其反序列化后载入到内存数据库中。


在初始化阶段，Leader会优先以全量同步方式来同步数据。同时，会根据Leader和Learner之间的数据差异情况来决定最终的数据同步方式。
