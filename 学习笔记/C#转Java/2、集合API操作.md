# 集合API操作

`List<A>` 转换为 `List<B>`，在java里也有类似C#里`IQueryable<out T>`，java里为`Stream<T>`。

```java
// 将List<OrderItem> 转为 List<OrderItemDTO>
List<OrderItemDTO> orderItemDtos = order.getItems().stream().map(new Function<OrderItem, OrderItemDTO>(){
        @Override
        public OrderItemDTO apply(OrderItem src) {
            return new OrderItemDTO(
                src.getGoodsCode(),
                src.getGoodsName(),
                src.getSku(),
                src.getQuantity());
        }
});

// Lambda表达式写法
List<OrderItemDTO> orderItemDtos = order.getItems().stream().map(src->new OrderItemDTO(
		src.getGoodsCode(),
		src.getGoodsName(),
		src.getSku(),
		src.getQuantity())
).collect(Collectors.toList());
```

