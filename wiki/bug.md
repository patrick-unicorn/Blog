+ 使用stream进行collection进行转换时，没有将转换后的Item添加到新的NewItemCollection中。

```
Collection newC = new Collection();
Collection oldC = find from datasource ...

if (oldC is not empty) {
    oldC.foreach(item -> {
        NewItem ni = transform(item);

        // miss this line
        newC.add(ni);
    })
}
```

