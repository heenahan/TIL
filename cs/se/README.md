# CS/SE

> 컴퓨터과학 중 하나인 소프트웨어 공학에 대한 공부한 내용입니다.

## CQS

Command Query Separation, 이하 CQS은 소프트웨어 디자인 패턴 중 하나이다.

모든 객체의 메서드를 command와 query 두 개로 구분하는 패턴이다. 이때 하나의 메서드가 command이면서 동시에 query일 수는 없다. 메서드는 둘 중 하나에만 속해야 한다.

command는 객체의 내부 상태를 바꾸지만 값을 반환하지는 않는다. 예를 들어 setter 메서드는 command이다.

반대로 query는 객체의 내부 상태를 바꾸지 않고 객체의 값을 반환하기만 한다. 예를 들어 getter 메서드는 query이다. 

- https://medium.com/@su_bak/cqs-command-query-separation-pattern-%E1%84%8B%E1%85%B5%E1%84%85%E1%85%A1%E1%86%AB-f701eabf8754