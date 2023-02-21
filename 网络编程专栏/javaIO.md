# JAVAIO使用

## InetAddress用法

~~~java
//使用getLocalHost获取InetAddress对象（本机）
InetAddress address = InetAddress.getLocalHost();
System.out.println(address.getHostAddress()); //返回本机IP
System.out.println(address.getHostName()); //输出本机计算机名称
~~~

~~~java
//使用getByName获取InetAddress对象（百度）
InetAddress address = InetAddress.getByName("www.baidu.com");
System.out.println(address.getHostAddress()); //返回百度IP
System.out.println(address.getHostName()); //输出百度域名
~~~

~~~java
//根据IP得到InetAddress对象
InetAddress address = InetAddress.getByName("14.215.177.39");
System.out.println(address.getHostAddress()); //返回163服务器的IP
System.out.println(address.getHostName()); //输出IP而不是域名，如果解析不了可能可能不允许解析
~~~

## UDP使用

### 简单使用

#### 发送端

~~~java
public class UdpClient {
    public static void main(String[] args) throws Exception {
        System.out.println("发送方启动中.....");
        //使用DatagramSocket 指定端口 创建发送端
        DatagramSocket client = new DatagramSocket(8888);
        //准备数据 转换成字节数组
        String data = "UDP传输";
        byte[] datas = data.getBytes();
        //封装DatagramPacket包裹 指定发送目的地
        DatagramPacket packet = new DatagramPacket(datas, 0, datas.length, new InetSocketAddress("localhost",9999));
        //发送包裹send(DatagramPacket p)
        client.send(packet);
        //释放资源
        client.close();
    }
}
~~~

#### 接受端

~~~java
public class UdpServer {
    public static void main(String[] args) throws Exception {
        System.out.println("接收方启动中.....");
        //使用DatagramSocket 指定端口 创建接受端接口
        DatagramSocket server = new DatagramSocket(9999);
        //准备接收容器 封装成DatagramPacket
        byte[] con = new byte[1024*60];
        DatagramPacket packet = new DatagramPacket(con, 0, con.length);
        //阻塞式接受
        server.receive(packet);
        //获取数据 获取长度
        byte[] datas = packet.getData();
        int len = packet.getLength();
        //打印
        System.out.println(new String(datas,0,len));
    }
}
~~~

### 基本类型传输

#### 发送端

~~~java
public class UdpTypeClient {
    public static void main(String[] args) throws Exception {
        System.out.println("发送方启动中.....");
        //使用DatagramSocket 指定端口 创建发送端
        DatagramSocket client = new DatagramSocket(8888);
        //准备数据 转换成字节数组
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        DataOutputStream dos = new DataOutputStream(new BufferedOutputStream(baos));
        dos.writeUTF("UDP传输");
        dos.writeInt(18);
        dos.writeBoolean(false);
        dos.writeChar('a');
        dos.flush();
        byte[] datas = baos.toByteArray();
        System.out.println(datas.length);
        //封装DatagramPacket包裹 指定发送目的地
        DatagramPacket packet = new DatagramPacket(datas, 0, datas.length, new InetSocketAddress("localhost",9999));
        //发送包裹send(DatagramPacket p)
        client.send(packet);
        //释放资源
        client.close();
    }
}
~~~

#### 接收端

~~~java
public class UdpTypeServer {
    public static void main(String[] args) throws Exception {
        System.out.println("接收方启动中.....");
        //使用DatagramSocket 指定端口 创建接受端接口
        DatagramSocket server = new DatagramSocket(9999);
        //准备接收容器 封装成DatagramPacket
        byte[] con = new byte[1024*60];
        DatagramPacket packet = new DatagramPacket(con, 0, con.length);
        //阻塞式接受
        server.receive(packet);
        //获取数据 获取长度
        byte[] datas = packet.getData();
        int len = packet.getLength();
        DataInputStream dis = new DataInputStream(new BufferedInputStream(new ByteArrayInputStream(datas)));
        String msg = dis.readUTF();
        int age = dis.readInt();
        boolean flag = dis.readBoolean();
        char ch = dis.readChar();
        //打印
        System.out.println(msg+"==>"+age);
        server.close();
    }
}
~~~

### 对象类型传输

#### 发送端

~~~java
public class UdpObjClient {
    public static void main(String[] args) throws Exception {
        System.out.println("发送方启动中.....");
        //使用DatagramSocket 指定端口 创建发送端
        DatagramSocket client = new DatagramSocket(8888);
        //准备数据 转换成字节数组
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(new BufferedOutputStream(baos));
        Employee emp = new Employee("lzj", 400);
        oos.writeObject(emp);
        oos.flush();
        byte[] datas = baos.toByteArray();
        //封装DatagramPacket包裹 指定发送目的地
        DatagramPacket packet = new DatagramPacket(datas, 0, datas.length, new InetSocketAddress("localhost",9999));
        //发送包裹send(DatagramPacket p)
        client.send(packet);
        //释放资源
        client.close();
    }
}
class Employee implements Serializable {
    private transient String name;
    private double salary;

    public Employee() {
    }

    public Employee(String name, double salary) {
        this.name = name;
        this.salary = salary;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public double getSalary() {
        return salary;
    }

    public void setSalary(double salary) {
        this.salary = salary;
    }

    @Override
    public String toString() {
        return "Employee{" +
                "name='" + name + '\'' +
                ", salary=" + salary +
                '}';
    }
}
~~~

#### 接受端

~~~java
public class UdpObjServer {

    public static void main(String[] args) throws Exception {
        System.out.println("接收方启动中.....");
        //使用DatagramSocket 指定端口 创建接受端接口
        DatagramSocket server = new DatagramSocket(9999);
        //准备接收容器 封装成DatagramPacket
        byte[] con = new byte[1024*60];
        DatagramPacket packet = new DatagramPacket(con, 0, con.length);
        //阻塞式接受
        server.receive(packet);
        //获取数据 获取长度
        ObjectInputStream ois = new ObjectInputStream(new BufferedInputStream(new ByteArrayInputStream(packet.getData())));
        Object employee = ois.readObject();
        if (employee instanceof Employee){
            Employee empObj = (Employee) employee;
            System.out.println(empObj.getName()+"==>"+((Employee) employee).getSalary());
        }
        //打印
        server.close();
    }
}
class Employee implements Serializable {
    private transient String name;
    private double salary;

    public Employee() {
    }

    public Employee(String name, double salary) {
        this.name = name;
        this.salary = salary;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public double getSalary() {
        return salary;
    }

    public void setSalary(double salary) {
        this.salary = salary;
    }

    @Override
    public String toString() {
        return "Employee{" +
                "name='" + name + '\'' +
                ", salary=" + salary +
                '}';
    }
}
~~~





