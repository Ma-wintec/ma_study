# Spring事务传播机制

## 前置工作

### 实体类

```java
@Data
@TableName("book")
public class Book {
    @TableId(value = "book_id",type = IdType.ASSIGN_ID)
    private String bookId;
    private String bookName;
}

@Data
@TableName("food")
public class Food {
    @TableId(value = "food_id", type = IdType.ASSIGN_ID)
    private String foodId;
    private String foodName;
}

@Data
@TableName("user")
public class User {
    @TableId(value = "user_id",type = IdType.ASSIGN_ID)
    private String userId;
    private String userName;
}
```

### 业务层

#### BookService

```java
@Service
public class BookService {
    @Resource
    private BookMapper bookMapper;

    public String addBook(String bookName) {
        Book book = new Book();
        book.setBookName(bookName);
        bookMapper.insert(book);
        return "插入book{" + book.getBookId() + "," + book.getBookName() + "}完成";
    }

    public List<Book> findBook() {
        return bookMapper.selectList(null);
    }

    public Book findBookById(String bookId) {
        return bookMapper.selectOne(new QueryWrapper<Book>().lambda().eq(Book::getBookId, bookId));
    }

    public String deleteBookById(String bookId) {
        bookMapper.deleteById(bookId);
        return "删除book{" + bookId + "}成功";
    }

    public String updateBook(Book book) {
        bookMapper.updateById(book);
        return "更新book{" + book.getBookId() + "," + book.getBookName() + "}成功";
    }
}
```

#### UserService

```java
@Service
public class UserService {
    @Resource
    private UserMapper userMapper;

    public User findUser(String userId) {
        return userMapper.selectById(userId);
    }

    @Transactional(propagation = Propagation.NEVER)
    public String addUser(String userName) {
        User user = new User();
        user.setUserName(userName);
        userMapper.insert(user);

        int a = 1/0;

        return "添加用户{" + userName + "}成功！";
    }

    public String deleteUserById(String userId) {
        userMapper.deleteById(userId);
        return "删除用户{" + userId + "}成功！";
    }

}
```

#### MainService

```java
@Service
public class MainService {
    @Resource
    private BookService bookService;
    @Resource
    private UserService userService;
    @Resource
    private FoodMapper foodMapper;

    @Transactional(propagation = Propagation.REQUIRED)
    public String addBookandUser(String bookName, String userName,String foodName) {
        bookService.addBook(bookName);
        Food food = new Food();
        food.setFoodName(foodName);
        foodMapper.insert(food);
        userService.addUser(userName);
        return "chenggong!";
    }
}
```

## 7种传播机制

### ***REQUIRED***

支持当前事务;如果不存在，就创建一个新的。如果是事务方法嵌套调用，标有required的方法会将外事务直接拿过来使用，如果此事内部出现异常回滚会使外事务方法也回滚

```java
@Transactional(propagation = Propagation.REQUIRED)
```

![image-20220124140342042](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220124140342042.png)

很简单，没什么好说的，默认也是此传播机制；

### ***SUPPORTS***

支持当前事务;如果不存在，则执行非事务。

```java
@Transactional(propagation = Propagation.SUPPORTS)
```

**和REQUIRED的不同点：**

![image-20220124141754836](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220124141754836.png)

### ***NOT_SUPPORTED***

> 执行非事务处理，如果存在的话，暂停当前事务。

看不懂，说人话就是：

* 明确指定被标识的方法不要用事务。外部事务方法调用此方法时，事务不会进到此方法(即把外部事务挂起)，直到此方法执行完后。

```java
@Transactional(propagation = Propagation.NOT_SUPPORTED)
```

![image-20220124141739925](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220124141739925.png)

### ***NEVER***

不支持当前的事务;如果当前事务存在，则抛出异常。

```java
@Transactional(propagation = Propagation.NEVER)
```

顾名思义，执行到该方法时，如果检测到外部有事务，直接抛异常。

![image-20220124142020399](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220124142020399.png)

异常：

![image-20220124142041828](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220124142041828.png)

### ***MANDATORY***

支持当前事务;如果不存在当前事务，则抛出异常。

**完全就是NEVER的反义词**

```java
@Transactional(propagation = Propagation.MANDATORY)
```

![image-20220124142334934](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220124142334934.png)

### ***NESTED***

* 在同一个service中嵌套, 如果已经存在外层事务，则nested不会开启新的事务，否则会开启nested的savepoint是不起作用的， 内层事务回滚会导致整个事务一同回滚
* 在**不同的service中嵌套**，如果已经存在外层事务，则nested同样不会开启新的事务，否则会开启但是nested的savepoint是起作用的，即：**内层事务回滚 只会影响内层事务，不会导致外层事务一同回滚**

```java
@Transactional(propagation = Propagation.NESTED)
```

这里对代码进行一些调整，用try/catch包裹userService.addUser

```java
@Transactional(propagation = Propagation.REQUIRED)
public String addBookandUser(String bookName, String userName,String foodName) {
    System.out.println(TransactionSynchronizationManager.getCurrentTransactionName());
    bookService.addBook(bookName);
    Food food = new Food();
    food.setFoodName(foodName);
    foodMapper.insert(food);
    try {
        userService.addUser(userName);
    }catch (Exception e){
        e.printStackTrace();
    }
    return "chenggong!";
}
```

![image-20220124145951060](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220124145951060.png)

### ***REQUIRES_NEW***

无论如何都创建一个新的事务来执行被标识的方法。

**一般局部数据操作一致性都用此方法。**

![image-20220124152940044](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220124152940044.png)

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
```

![image-20220124152338089](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220124152338089.png)

**我个人认为其与NESTED的区别**

> NESTED即便是嵌套Service，但本质上addUser与主事务仍是一个事务，同名，可以理解为在进入addUser时记录了一个快照，如果addUser抛出异常，主事务无需回滚。
>
> ![image-20220124152559973](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220124152559973.png)
>
> REQUIRES_NEW本质上是为子事务新建了一个事务，在进入子事务时实际上是另起了一个事务，如果addUser抛出异常，主事务无需回滚。
>
> ![image-20220124152738272](https://xiaoma9969.oss-cn-beijing.aliyuncs.com/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%E5%9B%BE%E7%89%87/image-20220124152738272.png)
>
> **但是，当主事务发生异常报错时**
>
> * NESTED会回滚addUser，因为本质上他们是一个事务，仅记录了一个快照
> * REQUIRES_NEW则无法回滚addUser，因为子事务已经提交