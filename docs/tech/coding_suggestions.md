

## 代码整洁之道

- 为什么要有这样一份文档？

  记录学习心得，成为一个更好的程序员。

- 为什么要关注代码的整洁？

  减少代码的混乱，提高生产效率，让代码看起来简单明了易读

- 简洁代码的核心是什么？

  减少重复代码，提高表达力，提早构建简单抽象。

下面是具体的建议

#### 命名

总体来说，命名应该做到**语义明确，名副其实，避免误导**（比如名称为``accounList``的变量实际上确是一个数组类型 ，或者两个变量名的拼写很相近）。

- case1

  ```java
    public List<int[]> getThme() {
          List<int[]> list1 = new ArrayList<>();
          for (int[] x : theList) {
              if (x[0] == 4) {
                  list1.add(x);
              }
          }
          return list1;
      }
  ```

  代码简洁，却很模糊：theList是什么东西？x[0]零下标的意义是什么？值4的意义？怎么使用返回的列表？

  这些问题的答案都没有体现在代码中。实际在开发中，比如theList代表扫雷游戏的盘面，那么应命名为``gameboard`` 。

  better one:

  ```java
   public List<Ceil> getFlaggedCells() {
          List<Ceil> flaggedCeills = new ArrayList<>();
          for (Cell cell : gameboard) {
              if (cell.isFlagged()) {
                  flaggedCeills.add(cell);
              }
          }
          return flaggedCeills;
      }
  ```

  上述方法中另写了一个类Ceil，包装了数组，并且用``isFlagged``掩盖住了魔术数4。语义更明确了。

- case2 

  ```java
   private void printGuessStatistics(char candidate, int count) {
          String number;
          String verb;
          String pluralModifier;
          if (count == 0) {
              number = "no";
              verb = "are";
              pluralModifier = "s";
          } else if (count == 1) {
              number = "1";
              verb = "is";
              pluralModifier = "";
          } else {
              number = Integer.toString(count);
              verb = "are";
              pluralModifier = "s";
          }
          String message = String.format("There %s %s %s%s", verb, number, candidate, pluralModifier);
          System.out.println(message);
      }
  ```

  上述函数有点长，变量的使用贯穿始终。更好的做法是，分解这个函数，创建一个``GuessStatisticsMesaage``的类，将这个三个变量定义为成员变量。让方法分解为更小的函数看起来更加干净利落。

  better one：

  ```java
  public class GuessStatisticsMesaage {
      private String number;
      private String verb;
      private String pluralModifier;
      public String make(char candidate, int count) {
          createPluralDependentMessageParts(count);
      return String.format("there %s %s %s%s", verb, number, candidate, pluralModifier);
      }
  
      private void createPluralDependentMessageParts(int count) {
          if (count == 0){
              thereAreNoLetters();
          } else if (count == 1){
              thereIsOneLetter();
          } else {
              thereAreManyLetters();
          }
      }
    // 实现对应thereAreNoLetters、thereIsOneLetter、thereAreManyLetters方法
  }
  ```

#### 函数

函数应该短小，只做一件事情，尽量保证函数中的语句在**同一抽象层级**上。

- 如何写出这样的函数？

先想什么就写什么，配上一套单元测试，覆盖每行丑陋的代码。然后再打磨，分解函数，修改名称，消除重复。



























#### 容易忽视的编码规范

- <font color=red>【强制】</font>>类型为布尔型的类字段，禁止使用`is`开头，以防部分框架解析和序列化错误。
  <font color=blue>说明</font>：不同框架、序列化器可能遵循不同的规范。例如一个名为`isDeleted`的布尔型属性，它的访问方法也是`isDeleted()`，框架在反向解析时可能误认为对应的属性名是`deleted`，导致属性访问不到。

- <font color=red>【强制】</font>为了保持方法的简洁，一个方法中传入的参数不能超过7个；如一个方法需传入大量参数，可考虑使用命令模式。
  <font color=blue> 说明</font>：过多的传入参数可能意味着一个方法缺乏封装，或者承担了过多职责需要拆分；对于构造方法需要传入很多参数的情景，可以考虑使用Builder模式构造这样的复杂对象

- <font color = red>【强制】</font>不允许任何魔法值（即未经预先定义的常量）直接出现在代码中。
  <font color = blue>说明</font>：所谓魔法值是指有特殊业务含义的值，例如超时时间、默认值等。作为表示数学运算含义的数值，通常不认为是魔法值，包括单位转换、位移位数、0值判断等，无需定义为常量。在代码格式检查中，我们也放行了一些常见数值。
  正例：MAX_HTTP_TIMEOUT_SECOND = 5、DEFAULT_PAGE_SIZE = 20
  反例：NUMBER_ZERO = 0（不要这么写，直接使用0就可以了）
- <font color = red>【强制】</font>在long或者Long赋值时，数值后使用大写字母L，不能是小写字母l，小写容易跟数字混淆，造成误解。

### 一些习惯
##### project相关

1.需要用到其他系统服务的时候，需要考虑一个问题：如果这个系统挂掉了，会对我的系统什么影响？这关系到是否抛出异常，阻断这个方法的执行，还是接着往下走。如果不阻断，就不要抛异常，流程接着往下走，一般是其他系统的这个方法调用不关系到数据一致性相关的问题。如果和是数据更新有关的方法，一般都需要抛异常，不让流程接着往下走。

2.对系统进行修改，尽量新增，而不是修改，待稳定了一段时间后，再慢慢废弃原先的代码/配置即可。因为要考虑到一些上线后遇到问题的回滚情况。

比如，我修改了kconf的配置，把灰度城市的文件id由fileid替换成了http地址，一旦上线后需要回滚，还必须重新修改kconf配置文件，把http替换为filedID。因此不如直接在配置中直接添加一个http的字段，根据逻辑获取。这样就算回滚，不会影响原先的代码。

还有个就是，比如发送消息功能，全部迁移到消息系统，如果出现问题，比如样式变化， 或者其他系统导致的消息不能发送，应该能支持迅速切换为原先的发送消息的模式。因此不应该将原先的不用代码删除，而是保留，通过一个开关或者 if else 进行消息发送的控制。相当于需要一个兜底策略。

3. 自己写方法的时候，方法名最好能够清楚地反应这个方法是在干什么，提高后续开发的效率，还能减少bug率。 比如入职系统中，很多方法做的事情太多了，以为只是一个简单的判断方法，结果还做了一些列同步数据的操作。


### 参考

- 快手编码规范
- 《代码整洁之道》
