[<< 返回到上页](index.md)

**这里将介绍服务端业务封装的博客文章**  

此文基于java语言，使用netty网络库进行业务封装的骨架说明。  

[1. 全局配置](#1)  
[2. 命令行指令](#2)  
[3. 网络IP配置](#3)  
[4. 协议处理](#4)  

<span id="1"></span>
## **1. 全局配置**  

在做服务端开发之前，需要想清楚一些全局数据需要单独写在配置文件里。这部分数据是必要的且容易变化(基本上是固定的)且开发人员环境不同导致数据不同等等原因。可以理解成数据驱动，业务逻辑与数据值本身无关。那么配置这些数据，在程序运行时读取解析使用就显得非常必要了。有一部分原因是hard code这样的硬编码不利于后期数据变化导致重新打包等因素。  
在这里罗列一些基础的需要配置的数据：  

```
# 连接池配置文件
bonecp_config=./res/bonecp_config.xml
# 命令所在包名,动态加载
command=com.kl.command.impl
# 日志记录配置文件
log=./res/log4j.properties
# http 资源路径
http_res=./res
# 服务端ip和监听端口
ip_port=0:172.10.13.44:9898,1:172.10.13.44:8080
# 线程池的大小
thread_pool_size=400
# 是否开启数据库
openDb=0
...
...
```
配置信息即为参数信息，在java开发中使用Properties数据结构，类似于Map表。  
配置文件解析，加载到内存：  
```
public class MJPathConfig
{
    public static Properties mProperties = null;
    private static ReadWriteLock mLock = new ReentrantReadWriteLock(true);
    private static String mPath;

    public static boolean initConfig(String path)
    {
        if (path == null || path.equals(""))
        {
            return false;
        }

        mPath = path;
        if (mProperties == null)
        {
            return loadProperties(path);
        }
        return true;
    }

    public static boolean refreshProperties()
    {
        if (mPath == null || mPath.isEmpty())
            return false;

        try
        {
            Properties temp = new Properties();
            InputStream inputStream = new BufferedInputStream(
                    new FileInputStream(mPath));
            temp.load(inputStream);

            mLock.writeLock().lock();
            mProperties = temp;
        }
        catch (FileNotFoundException e)
        {
            return false;
        }
        catch (IOException e)
        {
            return false;
        }
        finally
        {
            mLock.writeLock().unlock();
        }
        return true;
    }

    private static boolean loadProperties(String path)
    {
        try
        {
            mLock.writeLock().lock();
            mProperties = new Properties();
            InputStream inputStream = new BufferedInputStream(
                    new FileInputStream(path));
            mProperties.load(inputStream);
            return true;
        }
        catch (FileNotFoundException e)
        {
            return false;
        }
        catch (IOException e)
        {
            return false;
        }
        finally
        {
            mLock.writeLock().unlock();
        }
    }

    public static String getValue(String key)
    {
        if (mProperties == null)
        {
            return null;
        }
        String value = mProperties.getProperty(key);
        if (null == value)
            return null;
        try
        {
            return new String(value.getBytes(), "utf-8");
        }
        catch (UnsupportedEncodingException e)
        {
            return null;
        }
    }

    public static Object setValue(String key, String value)
    {
        return mProperties.put(key, value);
    }
    public static boolean checkValue(String key)
    {
        String ips = MJPathConfig.getValue("AdminIP");
        if (ips != null && key != null && !key.isEmpty())
        {
            for (String s : ips.split("\\|"))
            {
                if (s.equals(key))
                    return true;
            }
        }
        return false;
    }
}
```

<span id="2"></span>
## **2. 命令行指令**  

服务端开启后，有时需要在控制台窗口进行操作，则提供一些必要的命令行参数操作服务端。某些时候，这块的功能可以提供一个客户端的形式，通过点击事件以http的形式请求服务端处理，效果类似。  

对这个业务逻辑进行封装：  
1. 控制台参数输入并解析  
2. 解析完后，找出对应的命令处理，同时参数以需要的格式传递  
3. 外部调用启动接口，注册所有命令的响应处理实例并完成控制台响应  

```
public class CmdlineType
{
    public final static String TEST = "test";
    public final static String TOTLE = "total";
}

public class CmdParameters
{
    private String mName;
    private List<String> mValues = new ArrayList<>();
    
    //value形式为：v1 v2 v3
    public CmdParameters(String name, String value)
    {
        this.mName = name;
        String[] temps = value.split(" ");
        for (String item : temps)
        {
            item = item.trim();
            mValues.add(item);
        }
    }
    
    //value的形为 以空格分隔的一串字符串，第一个字符为 name，后面的为参数
    public CmdParameters(String value)
    {
        String[] temps = value.split(" ");
        if (temps.length > 0)
        {
            this.mName = temps[0].trim();
        }
        for (int i = 1; i < temps.length; i++)
        {
            mValues.add(temps[i].trim());
        }
    }

    public String name()
    {
        return mName;
    }

    public List<String> values()
    {
        return mValues;
    }

}

public abstract class CmdlineCommand
{
    private String mCmdLineType;
    private String mDescription;
    private List<CmdParameters> mParameters;
    
    public CmdlineCommand(String type, String des)
    {
        mParameters = new ArrayList<>();
        this.mCmdLineType = type;
        this.mDescription = des;
    }
    
    public final void execute() throws Exception
    {
        System.out.println("命令: " + this.mCmdLineType + " description: " + mDescription + " 运行开始.");
        executeBefore();
        executeRunning();
        executeAfter();
        System.out.println("命令: " + this.mCmdLineType + " description: " + mDescription + " 运行完成.");
    }

    //对参数进行验证等功能，在执行函数之前执行
    protected abstract void executeBefore() throws Exception;
    //对命令进行执行的函数
    protected abstract void executeRunning() throws Exception;
    //命令执行完毕之后的处理
    protected abstract void executeAfter() throws Exception;

    public String cmdLineType()
    {
        return mCmdLineType;
    }

    public String description()
    {
        return mDescription;
    }

    public List<CmdParameters> parameters()
    {
        return mParameters;
    }

    public void addParameter(String value)
    {
        this.mParameters.add(new CmdParameters(value));
    }
}

public class CmdlineServices
{
    private static Logger mLogger = LogManager.getLogger(CmdlineServices.class);
    private HashMap<String, Class<? extends CmdlineCommand>> mCommands = new HashMap<>();
    private static CmdlineServices instance = new CmdlineServices();

    private CmdlineServices()
    {
        initCommands();
    }

    public static CmdlineServices instance()
    {
        return instance;
    }

    private void initCommands()
    {
        mCommands.put(CmdlineType.TOTLE, TotalCommandLine.class);
    }

    public void print() throws InstantiationException, IllegalAccessException
    {
        for (Entry<String, Class<? extends CmdlineCommand>> entry : mCommands.entrySet())
        {
            String str = entry.getKey();
            Class<? extends CmdlineCommand> cmdClazz = entry.getValue();
            CmdlineCommand cmd = cmdClazz.newInstance();
            System.out.println("命令行  输入 参数 ：" + str + "  --  " + cmd.description());
        }
    }
    
    public void startCmdline()
    {
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));

        String cmd;
        
        while (true)
        {
            try
            {
                System.out.print("> ");
                cmd = br.readLine();
                cmd = cmd == null ? cmd : cmd.trim();
            
                if (cmd == null || cmd.equals(""))
                {
                    continue;
                }

                String[] temp = cmd.split("-");
                String type = temp[0].trim();
                type = type.toUpperCase();
                if (type.isEmpty() || type.contains(" "))
                {
                    mLogger.info("命令类型只支持单字命令");
                    continue;
                }
                executeCmd(type, temp);
            }
            catch (IOException e)
            {
                e.printStackTrace();
            }
        }

    }

    private void executeCmd(String type, String[] temp)
    {
        try
        {
            Class<? extends CmdlineCommand> clazz = mCommands.get(type);
            if (clazz == null)
            {
                mLogger.info("命令: " + type + " 不存在");
                return;
            }
            CmdlineCommand command = clazz.newInstance();
            for (int i = 1; i < temp.length; i++)
            {
                command.addParameter(temp[i].trim());
            }
            command.execute();
        }
        catch (Exception e)
        {
            e.printStackTrace();
        }
    }
}
```

外部调用接口,即开启响应控制台：  
```
CmdlineServices.instance().startCmdline();
```
这里有几个类需要说明：  
1. CmdlineCommand: 所有命令行命令都需要继承此类  
2. CmdlineServices 
    1. 提供外部调用接口，开启命令行响应  
    2. 注册命令行命令处理类，需要继承CmdlineCommand  
3. CmdParameters: 按照固定的格式，对控制台参数进行解析后，创建协议参数  
4. CmdlineType: 左右命令行命令的名称  

<span id="3"></span>
## **3. 网络服务封装**  

对于最核心重要的，莫过于网络库的封装，这里是基于Netty的封装：  
需要准备：  
1. 服务端绑定的ip和端口号  
2. 主线程和工作线程的初始化  
3. IO模式的选择  
4. 通道选项设定(长连接，tcp nodelay，reuseaddr等)  
5. 接收和发送数据的处理器设定  

根据Netty官方提供的测试代码以及修改：  
```
    public void Run()
    {
        // 主线程池
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        // 工作线程池
        EventLoopGroup workerGroup = new NioEventLoopGroup(8);
        try
        {
            ServerBootstrap b = new ServerBootstrap();
            // 添加服务的配置信息
            b.group(bossGroup, workerGroup)
             .channel(NioServerSocketChannel.class)
             .option(ChannelOption.SO_BACKLOG, 1024)
             .option(ChannelOption.SO_REUSEADDR, true)
             .option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
             .childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
             .childOption(ChannelOption.TCP_NODELAY, true) 
             .childOption(ChannelOption.SO_KEEPALIVE, true)
             //.handler(new LoggingHandler(LogLevel.INFO))
             .childHandler(new MJSocketChannelInitializer()); 
            // Start the server.
            mServerChannel = b.bind(mIP, mPort).sync().channel();
            // Wait until the server socket is closed.
            mServerChannel.closeFuture().sync();
        }
        catch (Exception e)
        {
            mLogger.debug("server: " + e.getMessage());
        }
        finally 
        {
            // Shut down all event loops to terminate all threads.
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
```
java中nio成为new io，与标准IO有几点不同：  
1. 标准的IO基于字节流和字符流进行操作的，而NIO是基于通道（Channel）和缓冲区（Buffer）进行操作，数据总是从通道读取到缓冲区中，或者从缓冲区写入到通道中。  
2. NIO可以让你非阻塞的使用IO，例如：当线程从通道读取数据到缓冲区时，线程还是可以进行其他事情。当数据被写入到缓冲区时，线程可以继续处理它。从缓冲区写入通道也类似。  
3. NIO引入了选择器select的概念，选择器用于监听多个通道的事件（比如：连接打开，数据到达）。因此，单个的线程可以监听多个数据通道。  
注意：nio实质上是对标准io的进一步封装。  
通道：对每一条socket连接，对应到Channel上，然后基于通道来操作socket  
缓冲区：socket接收和发送数据会先放到缓冲区内，这是一层封装。  
非阻塞IO: 用户线程向内核发起read()请求时，内核不管数据有没有都会有返回值。  
select: 内核提供的阻塞函数，用户线程将io操作的socket注册到select中，然后等待阻塞函数select返回。当数据到达后，socket被激活，select返回，用户可以进行read操作。  
```
{
// 注册
select(socket);
// 轮询
while(true) {
    // 阻塞
    sockets = select();
    // 数据到达, 解除阻塞
    for(socket in sockets) {
        if(can_read(socket)) {
        // 数据已到达, 那么socket阻不阻塞无所谓
　　　　　　　read(socket, buffer);
        process(buffer);
        }
    }
}
}
```
在IO几种模式上，一般选择IO多路复用。实际上, 我们可以给select注册多个socket, 然后不断调用select读取被激活的socket，实现在同一线程内同时处理多个IO请求的效果.我们把select轮询抽出来放在一个线程里, 用户线程向其注册相关socket或IO请求，等到数据到达时通知用户线程，则可以提高用户线程的CPU利用率.这样, 便实现了异步方式。  

// to do

处理器需要完成：  
1. 断包粘包处理  
2. 解压缩解密处理  
3. 压缩加密处理  
4. 组装成能够处理的协议包  

<span id="4"></span>
## **4. 协议处理**  

