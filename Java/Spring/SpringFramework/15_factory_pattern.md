# 1. 工厂模式与 Spring IoC
设计模式：一种可以被重复利用的解决方案
开发原则，是软件开发的依据，设计模式会尽量满足开发原则
GoF（Gang of Four）四人组，在《Design Patterns: Elements of Reusable Object-Oriented Software》（即《设计模式》一书）中描述了23种设计模式，所以一般情况下，GoF就是指23种设计模式
我们平常所说的设计模式就是指《设计模式》一书中描述的23种设计模式
除了GoF23种设计模式之外，还有其它的设计模式，比如：JavaEE的设计模式（DAO模式、MVC模式等）。
GoF23种设计模式可以分为三类：
创建型（5个）：解决对象创建问题。
其中包含单例模式、工厂方法模式、抽象工厂模式、建造者模式、原型模式
结构型（7个）：一些类或对象组合在一起的经典结构。
行为型（11个）：解决类或对象之间的交互问题。
工厂模式是解决对象创建问题的，所以工厂模式属于创建型设计模式。
这是因为Spring框架底层使用了大量的工厂模式。


# 1. 工厂模式的三种形态
简单工厂模式（Simple Factory）：不属于23种设计模式之一。
	简单工厂模式又叫做：静态工厂方法模式。
	简单工厂模式是工厂方法模式的一种特殊实现。
工厂方法模式（Factory Method）：是23种设计模式之一。
抽象工厂模式（Abstract Factory）：是23种设计模式之一。

# 2. 简单工厂模式
```
Spring中的BeanFactory就使用了简单工厂模式。
在简单工厂模式中，工厂类中生产产品的方法为静态方法
简单工厂模式的角色包括三个：
	抽象产品 角色
	具体产品 角色
	工厂类 角色
	
抽象产品角色
/**
 * ClassName: Weapon
 * Package: simple.cw.spring
 * Description:
 * 抽象产品角色 - 武器抽象类
 *
 * @Author tcw
 * @Create 2023-05-23 20:52
 * @Version 1.0
 */
public abstract class Weapon {
    
    /**
     * 武器的攻击方法
     */
    public abstract void attack();
    
}

具体产品角色
/**
 * ClassName: Tank
 * Package: simple.cw.spring
 * Description:
 * 具体产品角色 - 坦克类
 *
 * @Author tcw
 * @Create 2023-05-23 20:55
 * @Version 1.0
 */
public class Tank extends Weapon{
    @Override
    public void attack() {
        System.out.println("开炮!!!!!!");
    }
}

/**
 * ClassName: Fighter
 * Package: simple.cw.spring
 * Description:
 * 具体产品角色 - 飞机类
 *
 * @Author tcw
 * @Create 2023-05-23 20:56
 * @Version 1.0
 */
public class Fighter extends Weapon {
    @Override
    public void attack() {
        System.out.println("投掷炸弹....");
    }
}

/**
 * ClassName: Dagger
 * Package: simple.cw.spring
 * Description:
 * 具体产品角色 - 匕首类
 *
 * @Author tcw
 * @Create 2023-05-23 20:56
 * @Version 1.0
 */
public class Dagger extends Weapon{
    @Override
    public void attack() {
        System.out.println("砍！砍！砍！砍！");
    }
}

工厂类角色
由于简单工厂模式中的工厂类中生产产品的方法为静态方法，所以简单工厂模式又称为静态工厂方法模式
/**
 * ClassName: WeaponFactory
 * Package: simple.cw.spring
 * Description:
 * 工厂类角色 - 武器工厂
 *
 * @Author tcw
 * @Create 2023-05-23 20:58
 * @Version 1.0
 */
public class WeaponFactory {
    
    /**
     * 根据需要的武器类型创建武器
     *
     * @param weaponType 需要的武器类型
     * @return 需要的武器类型相应的武器对象
     */
    public static Weapon getWeapon(String weaponType) {
        // 将要返回的武器对象
        Weapon weapon = null;
        // 根据需要的武器类型创建武器
        switch (weaponType) {
            case "tank":
                weapon = new Tank();
                break;
            case "fighter":
                weapon = new Fighter();
                break;
            case "dagger":
                weapon = new Dagger();
                break;
        }
        return weapon;
    }
}

客户端程序
对于客户端来说，无需了解产品的具体生产过程，只需直接利用工厂角色进行产品的生产即可
由于客户端无需关心产品的具体生产过程，客户端只需要负责进行消费即可，工厂角色只需根据客户端的需求生产相应的产品即可，即客户端只负责消费，工厂角色只负责生产，使得生产者和消费者分离了
简单工厂模式的作用：使生产者和消费者的职责分离，对方都无需关心对方的具体细节
@org.junit.Test
public void test01() {
    // 利用工厂角色生产产品（生产者生产产品）
    Weapon tank = WeaponFactory.getWeapon("tank");
    Weapon fighter = WeaponFactory.getWeapon("fighter");
    Weapon dagger = WeaponFactory.getWeapon("dagger");
    // 使用武器进行攻击（消费者消费产品）
    tank.attack();
    fighter.attack();
    dagger.attack();
}

简单工厂模式解决的问题（优点）
客户端程序不需要关心对象的创建细节，需要哪个对象时，只需要向工厂索要即可，初步实现了责任的分离。
客户端只负责“消费”，工厂负责“生产”，生产和消费分离。

简单工厂模式的缺点
简单工厂模式不符合OCP开闭原则，因为在系统需要进行扩展时，需要修改原先已经写好的工厂类
工厂类的责任比较重大，不能出现任何问题，因为这个工厂类负责所有产品的生产，称为全能类，或者有人把它叫做上帝类。这个工厂类一旦出问题，整个系统必然全部瘫痪。（不要把所有鸡蛋放到一个篮子里面）
```

# 3. 工厂方法模式
```
工厂方法模式，区别与简单工厂模式，就是一个产品对应一个工厂类
工厂方法模式的角色包括：
抽象工厂角色
具体工厂角色
抽象产品角色
具体产品角色

抽象产品角色
/**
 * ClassName: Weapon
 * Package: simple.cw.spring
 * Description:
 * 抽象产品角色 - 武器抽象类
 *
 * @Author tcw
 * @Create 2023-05-23 20:52
 * @Version 1.0
 */
public abstract class Weapon {
    
    /**
     * 武器的攻击方法
     */
    public abstract void attack();
    
}

具体产品角色
/**
 * ClassName: Tank
 * Package: simple.cw.spring
 * Description:
 * 具体产品角色 - 坦克类
 *
 * @Author tcw
 * @Create 2023-05-23 20:55
 * @Version 1.0
 */
public class Tank extends Weapon{
    @Override
    public void attack() {
        System.out.println("开炮!!!!!!");
    }
}

/**
 * ClassName: Fighter
 * Package: simple.cw.spring
 * Description:
 * 具体产品角色 - 飞机类
 *
 * @Author tcw
 * @Create 2023-05-23 20:56
 * @Version 1.0
 */
public class Fighter extends Weapon {
    @Override
    public void attack() {
        System.out.println("投掷炸弹....");
    }
}

/**
 * ClassName: Dagger
 * Package: simple.cw.spring
 * Description:
 * 具体产品角色 - 匕首类
 *
 * @Author tcw
 * @Create 2023-05-23 20:56
 * @Version 1.0
 */
public class Dagger extends Weapon{
    @Override
    public void attack() {
        System.out.println("砍！砍！砍！砍！");
    }
}

抽象工厂角色
/**
 * ClassName: WeaponFactory
 * Package: simple.cw.spring
 * Description:
 * 抽象工厂角色 - 武器工厂
 *
 * @Author tcw
 * @Create 2023-05-23 20:58
 * @Version 1.0
 */
public abstract class WeaponFactory {
    
    /**
     * 生产武器
     *
     * @return 武器
     */
    public abstract Weapon getWeapon();
}

具体工厂角色
/**
 * ClassName: DaggerFactory
 * Package: simple.cw.spring
 * Description:
 * 具体工厂角色 - 生产匕首的工厂
 *
 * @Author tcw
 * @Create 2023-05-24 18:15
 * @Version 1.0
 */
public class DaggerFactory extends WeaponFactory {
    @Override
    public Weapon getWeapon() {
        return new Dagger();
    }
}

/**
 * ClassName: TankFactory
 * Package: simple.cw.spring
 * Description:
 * 具体工厂角色 - 生产坦克的工厂
 *
 * @Author tcw
 * @Create 2023-05-24 18:18
 * @Version 1.0
 */
public class TankFactory extends WeaponFactory{
    @Override
    public Weapon getWeapon() {
        return new Tank();
    }
}

/**
 * ClassName: FighterFactory
 * Package: simple.cw.spring
 * Description:
 * 具体工厂角色 - 生产飞机的工厂
 *
 * @Author tcw
 * @Create 2023-05-24 18:19
 * @Version 1.0
 */
public class FighterFactory extends WeaponFactory{
    @Override
    public Weapon getWeapon() {
        return new Fighter();
    }
}

客户端程序
@org.junit.Test
public void test01() {
    // 创建工厂类对象
    WeaponFactory tankFactory = new TankFactory();
    WeaponFactory fighterFactory = new FighterFactory();
    WeaponFactory daggerFactory = new DaggerFactory();
    // 利用工厂对象生产产品
    Weapon tank = tankFactory.getWeapon();
    Weapon fighter = fighterFactory.getWeapon();
    Weapon dagger = daggerFactory.getWeapon();
    // 使用武器进行攻击
    tank.attack();
    fighter.attack();
    dagger.attack();
}

工厂方法模式解决的问题（优点）
工厂方法模式，由于一个产品对应一个工厂类，所以在新增产品时，只需要新增一个工厂类和一个具体产品类即可，不用修改原程序，客户端需要新的产品时，只需要调用新的工厂类的相应方法即可，解决了简单工厂模式违背OCP原则的问题；
扩展性高
同时由于一个产品对应一个工厂类，所以工厂也就不是全能类，避免了工厂类一旦出问题导致整个系统必然全部瘫痪的问题
一个调用者想创建一个对象，只要知道其名称就可以了，生产和消费分离。
屏蔽产品的具体实现，调用者只关心产品的接口。

工厂方法模式的缺点
每次增加一个产品时，都需要增加一个具体类和对象实现工厂，使得系统中类的个数成倍增加，在一定程度上增加了系统的复杂度，同时也增加了系统具体类的依赖。
类爆炸，由于类的个数急剧增多，类之间的关系复杂度也急剧上升，导致系统难维护
工厂方法模式缺点的解决，使用抽象工厂模式

```

# 4. 源码视角：Spring 内部如何应用工厂模式

## 4.1. BeanFactory — 简单工厂模式的 Spring 实现

```
// BeanFactory 是最典型的简单工厂：
// 根据 beanName 返回不同类型的对象

public interface BeanFactory {
    Object getBean(String name) throws BeansException;
    <T> T getBean(String name, Class<T> requiredType) throws BeansException;
    // ...
}

// 内部实现（AbstractBeanFactory.getBean()）：
// 1. 从 beanDefinitionMap 查找 BeanDefinition（产品的生产规则）
// 2. 调用 createBean() 创建产品
// 3. 返回产品实例

// 与标准简单工厂的对应关系：
// 工厂类：DefaultListableBeanFactory (implements BeanFactory)
// 静态工厂方法：不是 static，但语义等价 — getBean(String) → Object
// 产品族：所有注册的 Bean 都是产品
```

## 4.2. FactoryBean — 工厂方法模式的 Spring 实现

```
// FactoryBean 定义了 getObject() 工厂方法
// 每个 FactoryBean 实现类是一个具体工厂，生产一种特定产品

public interface FactoryBean<T> {
    T getObject() throws Exception;        // ★ 工厂方法
    Class<?> getObjectType();
    default boolean isSingleton() { return true; }
}

// Spring 内部的 FactoryBean 实例：
SqlSessionFactoryBean    → 工厂方法 getObject() → SqlSessionFactory
MapperFactoryBean<T>     → 工厂方法 getObject() → T (Mapper 代理对象)
ProxyFactoryBean         → 工厂方法 getObject() → AOP 代理对象
JndiObjectFactoryBean    → 工厂方法 getObject() → JNDI 查找对象
PropertiesFactoryBean    → 工厂方法 getObject() → Properties

// 与标准工厂方法模式的对应关系：
// 抽象工厂：FactoryBean<T> 接口
// 具体工厂：SqlSessionFactoryBean implements FactoryBean<SqlSessionFactory>
// 抽象产品：SqlSessionFactory
// 具体产品：DefaultSqlSessionFactory（由 getObject() 内部创建）
```

## 4.3. Spring 中 BeanFactory 和 FactoryBean 的协作

```
getBean("sqlSessionFactory") 的执行流程：

1. getBean("sqlSessionFactory")
     ↓
2. 查找 beanDefinitionMap
   → beanClass = SqlSessionFactoryBean (implements FactoryBean)
     ↓
3. createBean("sqlSessionFactory") 
   → new SqlSessionFactoryBean()  ← 创建工厂对象
   → populateBean() → 注入 dataSource, configLocation...
   → initializeBean() → afterPropertiesSet() → buildSqlSessionFactory()
     ↓
4. getObjectForBeanInstance(beanInstance, "sqlSessionFactory", ...)
   → 检查 beanInstance instanceof FactoryBean? → YES
   → getObjectFromFactoryBean(factoryBean, "sqlSessionFactory")
     → factoryBean.getObject()  ← ★ 调用工厂方法
     → 返回 DefaultSqlSessionFactory 实例
     ↓
5. 将 DefaultSqlSessionFactory 缓存到 factoryBeanObjectCache

★ 设计要点：
  - 如果想要获取 FactoryBean 本身，用 getBean("&sqlSessionFactory")
  - "&" 前缀告诉 Spring 不要调用 getObject()，直接返回工厂对象
```

## 4.4. 抽象工厂模式在 Spring 中的体现

```
Spring 中没有显式的抽象工厂接口，但 ApplicationContext 相关的工厂体系
体现了抽象工厂的思想：

AbstractApplicationContext（抽象工厂）
  ├── ClassPathXmlApplicationContext（具体工厂1：XML方式）
  ├── FileSystemXmlApplicationContext（具体工厂2：文件系统方式）
  ├── AnnotationConfigApplicationContext（具体工厂3：注解方式）
  └── GenericWebApplicationContext（具体工厂4：Web方式）

每个具体工厂创建的产品族是相同的：
  - BeanFactory（产品：DefaultListableBeanFactory）
  - BeanDefinitionReader（产品：XmlBeanDefinitionReader / AnnotatedBeanDefinitionReader）
  - ResourceLoader（产品：DefaultResourceLoader / ServletContextResourceLoader）
```
