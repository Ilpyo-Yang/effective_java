# 12장. 직렬화

## 아이템 85.자바 직렬화의 대안을 찾으라
#### 직렬화의 위험성을 회피하는 가장 좋은 방법은 아무것도 역직렬화하지 않는 것이다.
### 역직렬화 폭탄
```java
static byte[] bomb() {
    Set<Object> root = new HashSet<>();
    Set<Object> s1 = root;
    Set<Object> s2 = new HashSet<>();

    for (int i=0; i < 100; i++) {
        Set<Object> t1 = new HashSet<>();
        Set<Object> t2 = new HashSet<>();

        t1.add("foo"); // t1을 t2과 다르게 만든다.
        s1.add(t1); s1.add(t2);

        s2.add(t1); s2.add(t2);
        s1 = t1; s2 = t2;
    }
    return serialize(root);
}
```
+ deserialize(bomb())가 실행될 때, HashSet을 역직렬화 하기위해 2^100의 hashCode() 메서드가 호출된다.
+ 따라서 문제들을 대처하려면 애초에 신뢰할 수 없는 바이트 스트림을 역직렬화하는 일 자체를 없애야 한다.
### 직렬화의 대안
+ 크로스-플랫폼 구조화된 데이터 표현
  + JSON, Protocol Buffers(protobuf) 
  + 자바 직렬화의 위험성을 회피하면서 다양한 플랫폼 지원, 우수한 성능, 풍부한 지원 도구 등을 제공하는 다른 방식의 매커니즘 방식을 크로스-플랫폼 구조화된 데이터 표현이라 한다.
+ 객체 역직렬화 필터링(ObjectInputFilter)
  + 그래도 직렬화를 사용해야겠다면 java.io.ObjectInputFilter

## 아이템 86. Serializable을 구현할지는 신중히 결정하라
+ 어떤 클래스의 인스턴스를 직렬화할 수 있게 하려면 클래스 선언에 implements Serializable을 덧붙이면 된다.
### Serializable 구현을 신중하게 생각해야 하는 이유
+ 릴리스한 뒤에는 수정하기 어렵다.
+ 버그와 보안 구멍이 생길 위험이 높아진다는 점이다.
+ 해당 클래스의 신버전을 릴리스할 때 테스트할 것이 늘어난다는 점이다.
