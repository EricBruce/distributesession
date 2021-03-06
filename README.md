# 分布式Sesseion的实现

>1）  Session ID的共享
>Session ID是一个实例化Session对象的唯一标识，也是它在Web容器中可以被识别的唯一身份标签。Jetty和Tomcat容器会通过一个Hash算法，得到一个唯一的ID字符串，然后赋值给某个实例化的Session，此时，这个Session就可以被放入Web容器的SessionManager中开始它短暂的一生。在Servlet中，我们可以通过HttpSession的getId()方法得到这个值，但是我们无法改变这个值。当Session走到它一生尽头的时候，Web容器的SessionManager会根据这个ID将其“火化”。所以Session ID是非常重要的一个属性，并且要保证它的唯一性。在单系统中，Session ID只需要被自身的Web容器读写，但是在分布式环境中，多个Web容器需要共享同一个Session ID。因此，当某个子系统的Web容器产生一个新的ID时，它必须需要一种机制来通知其他子系统，并且告知新ID是什么。
>
>2）  Session中数据的复制
>和共享Session ID的问题一样，在分布式环境下，Session中的用户数据也需要在各个子系统中共享。当用户通过HttpSession的setAttribute()方法在Session中设置了一个用户数据时，它只存在于当前与用户交互的那个Web容器中，而对其他子系统的Web容器来说，这些数据是不可见的。当用户在下一步跳转到另外一个Web容器时，则又会创建一个新的Session对象，而此Session中并不包含上一步骤用户设置的数据。其实Session在分布式系统之间的复制实现是简单的，但是每次在Session数据发生变化时，都在子系统之间复制一次数据，会大大降低用户的响应速度。因此我们需要一种机制，即可以保证Session数据的一致性，又不会降低用户操作的响应度。
>
>3）  Session的失效
>Session是有生命周期的，当Session的空闲时间(maxIdle属性值)超出限制时，Session就失效了，这种设计主要是考虑到了Web容器的可靠性。当一个系统有上万人使用时，就会产生上万个Session对象，由于HTTP的无状态特性，服务器无法确切的知道用户是否真的离开了系统。因此如果没有失效机制，所有被Session占据的内存资源将永远无法被释放，直到系统崩溃为止。在分布式环境下，Session被简单的创建，并且通过某种机制被复制到了其他系统中。你无法保证每个子系统的时钟都是一致的，可能相差几秒，甚至相差几分钟。当某个Web容器的Session失效时，可能其他的子系统中的Session并未失效，这时会产生一个有趣的现象，一个用户在各个子系统之间跳转时，有时会提示Session超时，而有时又能正常操作。因此我们需要一种机制，当某个系统的Session失效时，其他所有系统的与之相关联的Session也要同步失效。
>
>4）  类装载问题
>在单系统环境下，所有类被装载到“同一个”ClassLoader中。我在同一个上打了引号，因为实际上并非是同一个ClassLoader，只是逻辑上我们认为是同一个。这里涉及到了JVM的类装载机制，由于这个主题不是本文的讨论重点，所以相关详情可以参考相关的JVM文档。因此即使是由memcached或是ZooKeeper返回的字节数组也可以正常的反序列化成相对应的对象类型。但是在分布式环境下，问题就变得异常的复杂。我们通过一个例子来描述这个问题。用户在某个子系统的Session中设置了一个User类型的对象，通过序列化，将User类型的对象转换成字节数组，并通过网络传输到了memcached或是ZooKeeper上。此时，用户跳转到了另外一个子系统上，需要通过getAttribute方法从memcached或是ZooKeeper上得到先前设置的那个User类型的对象数据。但是问题出现了，在这个子系统的ClassLoader中并没有装载User类型。因此在做反序列化时出现了ClassNotFoundException异常。
当然上面描述的4点挑战只是在实现分布式Session过程中面临的关键问题，并不是全部。其实在我实现分布式Session的整个过程中还遇到了其他的一些挑战。比如，需要通过filter机制拦截HttpServletRequest，以便覆盖其getSession方法。但是在不同的Web容器中（例如Jetty或是Tomcat）对HttpServletRequest的实现是不一样的，虽然都是实现了HttpServletRequest接口，但是各自又添加了一些特性在其中。例如，在Jetty容器中，HttpSession的实现类是一个保护内部类，无法从其继承并覆盖相关的方法，只能从其实现类的父类中继承更加抽象的Session实现。这样就会造成一个问题，我必须重新实现对Session整个生命周期管理的SessionManager接口。有人会说，那就放弃它的实现吧，我们自己实现HttpSession接口。很不幸，那是不可能的。因为在Jetty的HttpServletRequest实现类的一些方法中对Session的类型进行了强制转换（转换成它自定义的HttpSession实现类），如果不从其继承，则会出现ClassCastException异常。相比之下，Tomcat的对HttpServletRequest和HttpSession接口的实现还是比较标准的。由此可见，实现分布式Session其实是和某种Web容器紧密耦合的。并不像网上有些人的轻描淡写，仅仅覆盖setAttribute和getAttribute方法是行不通的。
>

<img src="http://thumbnail0.baidupcs.com/thumbnail/6ce537fd69e3de9f8592130405657fa0?fid=4044231730-250528-592223535158059&time=1463389200&rt=pr&sign=FDTAER-DCb740ccc5511e5e8fedcff06b081203-Kypwt%2fOV8Z7LR8z%2bGRNFqM8jyhY%3d&expires=8h&chkbd=0&chkv=0&dp-logid=3171542194485307352&dp-callid=0&size=c1600_u900&quality=90">



