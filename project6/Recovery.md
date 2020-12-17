# 개요

## 환경

```text
1. Ubuntu 18.04 (테스트)
2. Ubuntu 20.04 (실제 소스코드 작성)
3. g++ 7.5.0
4. C++ 17
```

## 컴파일 옵션

```text
-g -fPIC -I -std=c++17 -W -Wall -pthread
```

## 소스코드

```text
- enums.h
- useful.h
- lock.h
- file.cpp (file.h)
- page.cpp (page.h)
- file_manager.cpp (file_manager.h)
- buffer_manager.cpp (buffer.h, buffer_manager.h)
- bpt_node.cpp (bpt_node.h, bpt_node_impl.hpp)
- bpt.cpp (bpt.h)
- bptapi.cpp (bptapi.h)
- lock_manager.cpp (lock_manager.h)
- trx_manager.cpp (trx_manager.h)
- log_manager.cpp (log_manager.h, log.h)
- main.cpp (구현 반영 안됨)
```

</br>

# 구현 설명

결론적으로 General Lock Design, Recovery는 구현하지 못하였다.

새롭게 Log Manager를 구현하여, 해당 로그 매니저에서 로그 파일을 읽고, 로그에 새로운 LSN 값을 부여하고, 로그 버퍼를 관리하여 정책에 따라 flush 해주어 올바른 LSN 관계가 유지되도록 하였다. 또한, Update, Begin, Commit, Rollback 시에 로그를 각각 발급해주었으며, Xact가 abort 될 때 rollback 할 때에 ARIES 알고리즘에서의 undo pass와 비슷하게 작동하게 하였으며, next undo sequence number 또한 부여해주었다.

로그 타입은 이전 project5에서는 READ, UPDATE 두 가지가 있었는데, 이를 UPDATE, BEGIN, COMMIT, ROLLBACK, COMPENSATE 다섯 가지로 변경하였으며, READ에 대해서는 따로 로그를 발급하지 않았다.

로그 struct는 공통되는 부분과 달라지는 부분을 따로 선언하여 관리하였으며, 로그 버퍼는 내부적으로 std::vector로 구현하였다.