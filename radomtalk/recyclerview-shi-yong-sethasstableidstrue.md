# RecyclerView使用setHasStableIds(true)

### 解决更新时图片闪烁问题

当适配器更新时，即使是同一张图片重新加载也会闪烁。使用setHasStableIds(true)可解决；从源码角度来看，相当于我们平时给ImageView和图片做了一个tag绑定，检测到是url没变时，不再重新加载图片，也就不用重新计算、绘制，这样就避免了图片闪烁

### setHasStableIds(true)导致数据错乱，onbindviewholder不调用

重写recyclerview的getitemid

```kotlin
@Override
    public long getItemId(int position) {
        return position;
    }
```
