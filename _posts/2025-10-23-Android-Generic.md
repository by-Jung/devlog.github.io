---
title: "[Android] 제네릭(Generic) 이란?"
excerpt: "제네릭(Generic) 란?"

categories:
  - Android
tags:
  - [tag1, tag2]

permalink: /Android/Generic/

toc: true
toc_sticky: true

date: 2025-01-08
last_modified_at: 2026-04-05
---

# 제네릭(Generic)
> **클래스나 메서드에서 데이터 타입을 지정하지 않고, 사용할 때 타입을 결정할 수 있도록 하는 기능**

 - **코드 재사용성** 증가 → 여러 데이터 타입에 대해 하나의 클래스로 처리 가능  
 - **타입 안정성** 보장 → `ClassCastException` 방지  
 - **유연성** 증가 → 다양한 데이터 타입을 처리할 수 있음
	
## **제네릭 사용 예제**
``` java
public abstract static class Adapter<T, E extends ViewDataBinding>
        extends RecyclerView.Adapter<ViewHolder<E>> {
```
- `T` → 리스트에서 사용할 **데이터 타입** (예: `String`, `User`, `Product` 등)
- `E extends ViewDataBinding` → `ViewDataBinding`을 확장한 **데이터 바인딩 클래스**만 사용 가능 (타입 제한)   
- `ViewHolder<E>` → `E` 타입을 사용하는 ViewHolder


## **제네릭을 활용한 코드 설명**
- (1) **제네릭**을 사용하지 않는 경우
``` java
public class StringAdapter extends RecyclerView.Adapter<StringViewHolder> {
    private List<String> items;

    @Override
    public void onBindViewHolder(StringViewHolder holder, int position) {
        holder.bind(items.get(position));
    }
}
```
    - 만약 `Integer` 타입 리스트가 필요하면 `IntegerAdapter`를 따로 만들어야 함 → **중복 코드 발생**
    - 코드 중복 증가

- (2) **제네릭**을 사용하는 경우
```java
public abstract static class Adapter<T, E extends ViewDataBinding>
        extends RecyclerView.Adapter<ViewHolder<E>> {
    private List<T> mItems = new ArrayList<>();

    @Override
    public void onBindViewHolder(ViewHolder<E> holder, int position) {
        holder.onBindViewHolder(mItems.get(position));
    }
}
```
  - `T` → 어떤 데이터 타입이든 사용할 수 있도록 **유연성 증가**
  - `E` → **`ViewDataBinding`을 상속받는 클래스만 가능** (타입 안전성 확보)

# Android Framework / Hidden API
## 1. getSystemService 래핑
```java
public static <T> T getService(Context context, String name) {
    return (T) context.getSystemService(name);
}
```
- 사용
```
TelephonyManager tm = getService(context, Context.TELEPHONY_SERVICE);
```

## 2. Field 접근
- Field는 클래스 안에 **선언된 변수(멤버 변수)** 를 다루는 Reflection 객체
```java
public static <T> T getField(Object obj, String fieldName) throws Exception {
    Field field = obj.getClass().getDeclaredField(fieldName);
    field.setAccessible(true);
    return (T) field.get(obj);
}
```
  - `field.setAccessible(true)` : private라서 못 건드리는 것을 강제로 접근
  - `field.get(obj)` : bj 인스턴스 안에 들어있는 해당 필드 값을 가져옴

## 3. Method invoke
- Method는 클래스 안의 **메서드(함수)** 를 나타내는 Reflection 객체
- 메서드 자체를 꺼내서 실행
```java
public static <T> T call(Object obj, String methodName, Object... args) throws Exception {
    Method method = obj.getClass().getDeclaredMethod(methodName);
    method.setAccessible(true);
    return (T) method.invoke(obj, args);
}
```
  - `method.setAccessible(true)` : private 메서드 접근 가능하게 변경
  - `method.invoke(obj, args)` : 함수를 Reflection으로 실행