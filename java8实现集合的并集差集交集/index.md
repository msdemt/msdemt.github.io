# Java8实现集合的并集、差集、交集



使用java8 新特性 stream 实现 list 的并集、差集、交集

```java

    public static void main(String[] args) {
        List<String> list1 = new ArrayList<>();
        List<String> list2 = new ArrayList<>();
        list1.add("0.00");
        list1.add("0.03");
        list1.add("0.05");
        list1.add("0.06");
        list1.add("0.09");

        list2.add("0.00");
        list2.add("0.03");
        list2.add("0.05");
        list2.add("0.06");
        list2.add("0.08");

        System.out.println("---交集---");
        List list3 = list1.stream().filter(t -> list2.contains(t)).collect(Collectors.toList());
        list3.stream().forEach(System.out::println);

        System.out.println("---差集---");  
        List list4 = new ArrayList();
        if(list1.size() > list2.size()){
            list4 = list1.stream().filter(t -> !list2.contains(t)).collect(Collectors.toList());
        }else if(list1.size() < list2.size()){
            list4 = list2.stream().filter(t -> !list1.contains(t)).collect(Collectors.toList());
        }else{
            List list41 = list1.stream().filter(t -> !list2.contains(t)).collect(Collectors.toList());
            List list42 = list2.stream().filter(t -> !list1.contains(t)).collect(Collectors.toList());
            list4.addAll(list41);
            list4.addAll(list42);
        }
        list4.stream().forEach(System.out::println);

        System.out.println("---并集---");
        List list5 = new ArrayList();
        list5.addAll(list1);
        list5.addAll(list2);
        List list6 = (List) list5.stream().distinct().collect(Collectors.toList());
        list6.stream().forEach(System.out::println);
    }
```

输出结果：

```
---交集---
0.00
0.03
0.05
0.06
---差集---
0.09
0.08
---并集---
0.00
0.03
0.05
0.06
0.09
0.08
```

