# Redis实战

## 验证码业务

> 在很多的注册页面或者登录页面都会涉及到验证码登录，验证码注册，接下来我们就用redis实现这个功能，以下是业务需求
>
> 1. 输入手机号码，点击发送后随机生成6位数字，2分钟有效
> 2. 输入验证码，点击验证，返回失败或成功
> 3. 每个手机号每天只能输入3次

###  需求分析

> 根据以上需求分析得出需要实现如下功能
>
> 1. 生成随机6位数字 => 使用Java的Random类可完成
> 2. 验证码2分钟有效 => 把验证码放入到Redis中，设置120秒后过期
> 3. 判断验证码是否一致 => 接收前端发来的验证码与Redis中的比较
> 4. 每个手机每天只能发3次验证码 => incr 发送后+1，大于2时不在发送验证码

### 代码实现

~~~java
public class PhoneCode {

    //生成6位验证码
    public static String getCode(){
        Random random = new Random();
        StringBuilder builder = new StringBuilder();
        for (int i = 0; i < 6; i++) {
            builder.append(random.nextInt(10));
        }
        return builder.toString();
    }

    //记录每个手机每天只能发送3次，以及验证码放入到redis
    public static void verifyCode(String phone){
        Jedis jedis = new Jedis("localhost",6379);
        //拼接key
        String countKey = "verifyCode" + phone + ":count"; //手机发送次数key
        String codeKey = "verifyCode" + phone + ":code"; //验证码的key

        //记录每个手机每天只能发送3次
        String count = jedis.get(countKey);
        //没有发送次数
        if (count == null){
            //如果没有设置一个初始，并且设置过期时间
            jedis.setex(countKey,60*60*24,"1");
        } else if (Integer.parseInt(count) <= 2) {
            //发送次数+1
            jedis.incr(countKey);
        }else if (Integer.parseInt(count) > 2) {
            //发送了3次，不能再发送
            System.out.println("今天发送次数以及超过三次");
        }

        //将生成的验证码放到redis中
        String vcode = getCode();
        jedis.setex(codeKey,60*20,vcode);
        jedis.close();
    }

    //验证码校验功能
    public static void getRedisCode(String phone,String code){
        //从redis获取验证码
        Jedis jedis = new Jedis("localhost",6379);
        String codeKey = "verifyCode" + phone + ":code"; //验证码的key
        String redisCode = jedis.get(codeKey);
        //判断
        if (redisCode.equals(code)){
            System.out.println("成功");
        }else {
            System.out.println("失败");
        }
        jedis.close();
    }

    public static void main(String[] args) {
        //模拟验证码生成，与发送
        verifyCode("12345678910");
        //校验验证码
        getRedisCode("12345678910","796472");
    }
}
~~~

## 分布式锁(Spring-Boot)

> 由于Redis的多条命令并非原子性操作，所以若需要实现分布式锁那么需要编写一些脚本语句，在Redis中一个Lua脚本就是一次原子性操作

### 加锁

> spring-boot-starter-data-redis 2.x之后版本使用如下方法加锁

~~~java
//重入锁-尝试获取锁
//lock:为锁的key
//tag:锁的标签与Key是一起组成唯一标记，用于预防分布式下被其它服务解锁，通常使用UUID生成
//timeOut:超时时长
//timeUnit:设置时长单位
public boolean tryLock(String lock, String tag, int timeOut, TimeUnit timeUnit){
    return redisTemplate.opsForValue().setIfAbsent(lock, tag, timeOut, timeUnit);
}
~~~

> spring-boot-starter-data-redis 2.x之前版本使用如下方法加锁

~~~java
//重入锁-尝试获取锁
//lock:为锁的key
//tag:锁的标签与Key是一起组成唯一标记，用于预防分布式下被其它服务解锁，通常使用UUID生成
//timeOut:超时时长(秒)
public boolean tryLock(String lock, String tag, int timeOut){
    String script = "if (redis.call('GET',KEYS[1])) then\n" +
        "    return 1\n" +
        "else\n" +
        "    redis.call('SET',KEYS[1],ARGV[1])\n" +
        "    redis.call('EXPIRE',KEYS[1],ARGV[2])\n" +
        "    return 0\n" +
        "end";
    DefaultRedisScript<Long> redisScript = new DefaultRedisScript<>();
    redisScript.setScriptText(script);
    redisScript.setResultType(Long.class);
    Long res = (Long) redisTemplate.execute(redisScript, Arrays.asList(lock), tag,timeOut);
    return res == 0;
}
~~~

### 解锁

~~~java
//重入锁-解锁
//lock:为锁的key
//tag:锁的标签与Key是一起组成唯一标记，用于预防分布式下被其它服务解锁，通常使用UUID生成
public void unLock(String lock, String tag){
    //释放锁，使用lua脚本，原子性操作
    String script = "if redis.call('get', KEYS[1]) == ARGV[1] then " +
        "return redis.call('del', KEYS[1]) " +
        "else " +
        "return 0 " +
        "end";
    DefaultRedisScript<Long> redisScript = new DefaultRedisScript<>();
    redisScript.setScriptText(script);
    redisScript.setResultType(Long.class);
    redisTemplate.execute(redisScript, Arrays.asList(lock),tag);
}
~~~

### 使用案例

~~~java
public void testLock(){
    String tag = UUID.randomUUID().toString(); //生成uuid
    String lock = "lock:"+"业务的唯一标记"; //获取锁
    
    if (tryLock(lock,tag,30,TimeUnit.SECONDS)){
        try{
            //业务代码
        }finally {
            unLock(lock,tag)
        }
    }else {
        //获取锁失败后，每隔0.1秒再次获取
        try {
            Thread.sleep(100);
            testLock();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
~~~

