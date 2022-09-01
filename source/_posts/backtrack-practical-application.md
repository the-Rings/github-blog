---
title: 回溯在项目中的一次应用
date: 2022-07-20 11:17:49
categories:
- Algorithm
tags:
- Backtrack
---

某自动化测试的项目，对于某个接口的测试，事先准备好了测试用例。比如，HTTP调用某个API接口的URL，其中需要几个参数
```
name: "James", "$James", "123", "-1", ""
date: "2020-07-01", "2020/07/01", "2020.07", ""
is_admin: "false", "true", "", "/"
tags: "['stuff', '', 'manager']", "[]", ""
```
此时需要对以上4个参数进行随机组合，接口可以传递的参数个数是0-4个，每个参数随机选择其后取值范围中的一个值来模拟调用。
例如，可能的情况是有600种
1. `{name: "123", date: "2020-07-01", "is_admin": "true", tags: ""}`
2. `{name: "", date: "2020.07", is_admin: "false"}`
3. `{name: "James", date: "2020-07-01"}`
4. `...`
...
600. `{}`

以上问题可以抽象出一个模型出来，有m堆小球，每堆都有n个不同的小球，第一阶段先确定从哪几堆中取，第二阶段每堆取一个，一共有多少种取法，并把所有的组合罗列出来？

穷举问题可以使用“回溯”，第一阶段可以抽象为求集合的全部子集（0-1背包问题），第二阶段在第一阶段基础上确定了某个子集，一次对子集中对应堆的小球每个从中取一个（K阶段决策模型）
```java
import java.util.*;
public class Solution {
  /**
   * 有m个key，每个key有m个值，如何列出所有的k-v组合？
   * 例如：
   * {
   *    a: [1, 2, 3],
   *    b: [4, 5, 6],
   *    c: [7, 8, 9],
   *    d: [!, @, #],
   * }
   * 第一步：先获得参数名的子集
   * 第二步：根据某个参数名的组合（例如：{b, c}），从b对应的[4, 5, 6]中任选出一个值，从c对应的[7, 8, 9]中任选出一个值，
   * 拼出3*3=9中不同的组合输出
   */
  Map<String, List<String>> valuesMap = new HashMap<>();
  private List<List<String>> paramsSubset(List<String> keys) {
    List<List<String>> result = new ArrayList<>();
    backtrack_p(keys, 0, new ArrayList<>(), result);
    return result;
  }

  private void backtrack_p(List<String> keys, int k, List<String> path, List<List<String>> result) {
    if (k == keys.size()) {
      result.add(new ArrayList<>(path));
      return;
    }
    backtrack_p(keys, k+1, path, result);
    path.add(keys.get(k));
    backtrack_p(keys, k+1, path, result);
    path.remove(path.size()-1);
  }

  private List<Map<String, String>> assign(List<String> params) {
    List<Map<String, String>> result = new ArrayList<>();
    backtrack_v(params, 0, new ArrayList<>(), result);
    return result;
  }

  private void backtrack_v(List<String> params, int k, List<String> path, List<Map<String, String>> result) {
    if (k == params.size()) {
      Map<String, String> hashMap = new HashMap<>();
      for (int i = 0; i < path.size(); i++) {
        hashMap.put(params.get(i), path.get(i));
      }
      result.add(hashMap);
      return;
    }
    int n = valuesMap.get(params.get(k)).size();

    for (int i = 0; i < n; i++) {
      path.add(valuesMap.get(params.get(k)).get(i));
      backtrack_v(params, k+1, path, result);
      path.remove(path.size()-1);
    }
  }

  public List<Map<String, String>> combine(List<String> keys) {
    List<List<String>> paramResult = paramsSubset(keys);
    List<Map<String, String>> finalResult = new ArrayList<>();
    for (int i = 0; i < paramResult.size(); i++) {
      finalResult.addAll(assign(paramResult.get(i)));
    }
    return finalResult;
  }

  public static void main(String[] args) {
    Solution solution = new Solution();
    solution.valuesMap.put("a", Arrays.asList("1", "2", "3", "4", "5"));
    solution.valuesMap.put("b", Arrays.asList("4", "5", "6", "*"));
    solution.valuesMap.put("c", Arrays.asList("4", "5", "6", "*"));
    solution.valuesMap.put("d", Arrays.asList("!", "@", "#"));
    List<Map<String, String>> r = solution.combine(new ArrayList<>(solution.valuesMap.keySet()));
    System.out.println(r.toString());
    System.out.println(r.size());
  }
}
```
