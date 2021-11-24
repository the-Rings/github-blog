---
title: java-annotation
date: 2021-11-24 10:31:14
tags:
 - Annotation
---

通过格式化的方式让代码附带信息, 之后根据这些约定好的格式, 在源代码时期, 编译时期和运行时期, 通过Annotation Processor做一些操作. 注解可以理解为一种特殊的注释, 只不过是给机器看的.
定义注解的语法看起来就像在定义一个接口.

# Annotation
定义在注解中的类型包括: 基本类型(All primitives)、String、Class、enum、Annotation以及以上这些类型组成的数组。

注解不支持继承.

有五种元注解, 元注解用于注解其他注解. 其中@Target和@Retention是必须的. @Retention标识了注解在那个阶段起作用，参数SOURCE，CLASS，RUNTIME中的一个，其中SOURCE表示注解将会被编译器丢弃（忽略），CLASS表示注解将会被JVM丢弃（忽略），RUNTIME表示注解可以再运行时起作用，并一直保留，所以可以通过反射读取注解信息。也就是说，SOURCE在complie time起作用，javac编译完就丢弃，比如@SuppressWarnings @Override就用在启动编译的时候。比如:
有一个类库叫mapstruct，其中@Mapper注解用在接口上对应@Retentation(RetentationPolicy.CLASS)。在它编译打包后，会在target/generated-sources/annotations/下生成对应的实现类的java文件，然后生成实现类的class文件。

那么问题来了，SOURCE也是在编译时起作用，究竟在那个阶段呢？以下这张图展示了Java的编译过程：
{% plantuml %}
start

: your source code;

repeat: Parse and Enter;
    : ...;
repeat while (Annotation Processing)

: Analyse and Generate;
: 100100010;

end
{% endplantuml %}

在Annotation Processing的过程之后，RetentationPolicy.SOURCE就被丢弃了，同时RetentationPolicy.CLASS的注解要还保留在class文件中。


## Useing javac to Process Annotations
自定义注解之后，如果不对其进行处理，那么注解不会比注释更有用。

如果要在编译时期做一些事情(利用注解生成代码), 这时就需要用到`Annotation Processor`, 具体的做法是继承AbstractProcessor, 然后实现其process方法.

Annotation Processor机制是什么？.
如果在上一轮的processing中生成一个新的source code文件，那么这个文件将被再次检查annotation，直到没有新的source file产生。最后所有的源文件都被编译完成。这就是为什么mapstruct先生成java文件，然后生成class文件。

## Runtime Annotations Processing
在运行时通过反射完成一些操作的话，不是继承AbstractProcessor，只需要通过Java反射API中扩展的getAnnotation(xxx.class)方法即可。

比如，通过自定义注解@Query(...)来实现动态查询，事先在一个类Criteria，其中声明好查询条件的字段Fields，都用@Query装饰。然后写这样一个方法，接收Criteria对象criteria，criterai.getClass().getDeclaredFields()，遍历这个数组，然后对数组中的每个Field，field.setAccessible(true)，保证对private的访问。然后field.getAnnotation(Query.class)获得当前这个@Query传入的参数，确定查询条件是等于，like等等。根据filed.getName()获得字段名称，filed.get(criteria)获得对象中此属性的值。
```java
import cn.hutool.core.util.ObjectUtil;
import com.lee.annotation.Query;
import com.lee.config.mybatis.SimpleQueryWrapper;
import java.lang.reflect.Field;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import lombok.extern.slf4j.Slf4j;

/**
 * @author guomaoqing907
 */
@Slf4j
@SuppressWarnings({"unchecked", "all"})
public class QueryHelpForMybatis<T> {
  public static <T, Q> SimpleQueryWrapper<T> getQueryCondition(Q query) {
    List<SimpleQueryWrapper<T>> list = new ArrayList<SimpleQueryWrapper<T>>();
    SimpleQueryWrapper<T> qw = new SimpleQueryWrapper<>();
    if (query == null) {
      return qw;
    }
    try {
      List<Field> fields = getAllFields(query.getClass(), new ArrayList<>());
      for (Field field : fields) {
        boolean accessible = field.isAccessible();
        // 设置对象的访问权限，保证对private的属性的访
        field.setAccessible(true);
        Query q = field.getAnnotation(Query.class);
        if (q != null) {
          String propName = q.propName();
          String joinName = q.joinName();
          // 不支持join表的条件，需要在具体查询接口处另作处理
          if (!isBlank(joinName)) {
            continue;
          }
          String blurry = q.blurry();
          String attributeName = isBlank(propName) ? field.getName() : propName;
          boolean sortType = q.sortType();
          Class<?> fieldType = field.getType();
          Object val = field.get(query);
          if (ObjectUtil.isNull(val) || "".equals(val) || ObjectUtil.isEmpty(val)) {
            continue;
          }
          // 模糊多字段
          if (ObjectUtil.isNotEmpty(blurry)) {
            String[] blurrys = blurry.split(",");
            List<SimpleQueryWrapper<T>> conditionChain = new ArrayList<>();
            // NOTE: 如何实现 (a like '..' or b like '..' or c like '..') and d = '..'
            for (String s : blurrys) {
              qw.or().like(s, val.toString());
            }
            continue;
          }
          switch (q.type()) {
            case EQUAL:
              qw.eq(attributeName, val);
              break;
            case GREATER_THAN:
              qw.and(obj -> obj.ge(attributeName, val));
              break;
            case LESS_THAN:
              qw.and(obj -> obj.le(attributeName, val));
              break;
            case LESS_THAN_NQ:
              qw.and(obj -> obj.lt(attributeName, val));
              break;
            case INNER_LIKE:
              qw.and(obj -> obj.like(attributeName, val));
              break;
            case LEFT_LIKE:
              qw.and(obj -> obj.likeLeft(attributeName, val));
              break;
            case RIGHT_LIKE:
              qw.and(obj -> obj.likeRight(attributeName, val));
              break;
            case IN:
              List valList = (ArrayList) val;
              Object[] objArray = valList.toArray();
              qw.and(obj -> obj.in(attributeName, objArray));
              break;
            case NOT_EQUAL:
              qw.and(obj -> obj.ne(attributeName, val));
              break;
            case NOT_NULL:
              qw.and(obj -> obj.isNotNull(attributeName));
              break;
            case IS_NULL:
              qw.and(obj -> obj.isNull(attributeName));
              break;
            case BETWEEN:
              List<Object> between = new ArrayList<>((List<Object>) val);
              qw.and(obj -> obj.between(attributeName, between.get(0), between.get(1)));
              break;
            case SORT:
              List valListSort = (ArrayList) val ;
              Object[] objArraySort = valListSort.toArray();
              if(valListSort.toString().split(",").length<=2){
                Object object = "asc";
                if (objArraySort[1].equals(object)){
                   qw.orderBy(true, true, objArraySort[0].toString());
                }else {
                  qw.orderBy(true, false, objArraySort[0].toString());
                }
              }
              else {
                String[] stringsSort;
                for (Object objectSort : objArraySort){
                  stringsSort=objectSort.toString().split(",");
                  if (stringsSort[1].equals("asc")){
                    qw.orderBy(true, true, stringsSort[0]);
                  }else {
                    qw.orderBy(true, false, stringsSort[0]);
                  }
                }
              }
              break;
            default:
          }
        }
        field.setAccessible(accessible);
      }
    } catch (Exception e) {
      log.error(e.getMessage(), e);
    }
    return qw;
  }

  private static boolean isBlank(final CharSequence cs) {
    int strLen;
    if (cs == null || (strLen = cs.length()) == 0) {
      return true;
    }
    for (int i = 0; i < strLen; i++) {
      if (!Character.isWhitespace(cs.charAt(i))) {
        return false;
      }
    }
    return true;
  }

  public static List<Field> getAllFields(Class clazz, List<Field> fields) {
    if (clazz != null) {
      fields.addAll(Arrays.asList(clazz.getDeclaredFields()));
      getAllFields(clazz.getSuperclass(), fields);
    }
    return fields;
  }
}
```


