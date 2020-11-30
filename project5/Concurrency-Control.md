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
- main.cpp (구현 반영 안됨)
```

</br>

# Record Lock

## Lock Mode

Lock mode는 아래와 같이 enum class로 관리하였다.

```c++
enum class LockMode {
    SHARED = 0,
    EXCLUSIVE = 1
};
```

find와 update 함수에서 각각의 작동을 위해 lock을 acquire 하고자 하는데,  
read를 수행하는 find 함수에서는 shared lock을, write를 수행하는 update 함수에서는 exclusive 락을 획득하도록 하였다.

## Lock Manager에서의 Lock Table 구현

```c++
std::unordered_map<std::pair<int, int64_t>, lock_list_t> hash_table_;
```

Lock Table은 기본적으로 <table id, key(record)> pair를 키로 가지고 lock_list_t를 value로 가지는 해시 맵(STL의 unordered_map)으로 구현되어있다.  
lock_list_t는 lock_t 오브젝트를 링크드리스트로 관리하며, Lock Manager는
lock aquire 요청이 들어오면, 알맞은 lock_list_t에 lock_t object를 추가한다.

## Lock Acquire 과정

```text
1. lock_list_t에 아직 아무런 lock_t 오브젝트가 없다면 바로 acquire 해준다.

2. Shared Lock의 경우 
    A. 앞에 같은 trx id를 가진 Lock이 하나라도 있을 때 acquire 해준다.
    B. 바로 앞에 깨어있는 Shared Lock이 있을 때 acquire 해준다.

3. Exclusive Lock의 경우
    A. 앞에 같은 trx id를 가진 Exclusive Lock이 하나라도 있을 때 aquire 해준다.
    B. 앞에 같은 trx id를 가진 Shared Lock만 있을 때 acquire 해준다.
    C. 앞에 아무것도 없을 때 acquire 해준다.

4. 대기한다
    A. conditional variable 역할을 하는 can_wait_ 변수가 true로 설정되어있을 때 깨어나서 acquire 한다.
    B. Exclusive Lock의 경우 3-B, 3-C 조건을 체크하고 하나라도 만족하면 깨어나서 acquire 한다.
    C. Shared Lock의 경우 앞에 Shared Lock 들만 존재하는 경우 꺠어나서 acquire 한다.
```

acquire 하기 전에 Transaction Table에도 lock_t 오브젝트를 연결해주는데, 이때 해당 lock_t가 대기하는 lock_t의 trx id들을 같이 넘겨준다.  
자신이 대기하는 lock_t를 찾는 기준은 아래와 같다.

```text
1. Shared Lock의 경우
    A. 자신 앞에 있는 Exclusive Lock 하나

2. Exclusive Lock의 경우
    A. 자신 바로 앞에 있는 Exclusive Lock 하나
    또는
    B. 자신과 자신 앞에 있는 Exclusive Lock 사이의 모든 Shared Lock들
```

## Lock Release 과정

Lock Release는 기본적으로 Commit, 또는 Abort 시에만 호출되며,  
호출된 후에는 자신이 연결된 lock_list_t 오브젝트의 링크드리스트에서 자기 자신을 제거하고,  
링크드리스트를 차례대로 순회하며 자신 뒤에 있던 Shared Lock들을 다음 Exclusive Lock을 만나기 전까지 can_wake_ 변수를 true로 설정해준다.

# Deadlock Detection

## Wait-For-Graph

Deadlock Detection을 구현하기 위하여 Wait-for-graph를 구현하였다.

```c++
std::unordered_map<int, std::unordered_map<int, bool>> wait_for_graph_;
```

위와 같이 Adjacency Matrix로 구현하였다.  
unordered_map으로 구현한 이유는 vector나 array를 활용하면 시간 측면에서 이득은 있으나, transaction이 계속하여 생성되고 종료되는 바, 삭제와 삽입을 쉽게 하기 위해서는 unordered_map이 가장 적절하다고 생각하였기 때문이다.
</br></br>
wait-for-graph의 간선 삭제는 commit 또는 abort 시에 해당 trx id에 해당하는 key를 아예 삭제함으로써 구현해주었다.  
간선을 추가하는 작업은 Lock Acquire 시에 Transaction Manager의 메소드를 호출해 새로운 lock_t 오브젝트를 연결할 시에 받아오는 wait list를 통해 해 주었고, 간선 추가 이후에 바로 Deadlock Detection을 진행하였다.

## DFS

Deadlock Detection은 DFS 탐색을 통해 Cycle을 찾는 방식으로 구현하였다.  
새로 추가된 lock_t 오브젝트의 trx id에 대해서만 사이클을 찾아주면 되므로 모든 정점이 아닌, 해당 trx id 정점에 대해 한 번만 탐색하였다.

```c++
void TransactionManager::findCycle(int trx_id) {
    visited_[trx_id] = true;
    for (auto& i : transaction_table_) {
        if (wait_for_graph_[trx_id][i.first]) {
            if (!(visited_[i.first])) {
                findCycle(i.first);
            } else if (!(finished_[i.first])) {
                has_cycle_ = true;
            }
        }
    }
    finished_[trx_id] = true;
}
```

위와 같이 구현하였으며, 구글링을 통해 쉽게 찾을 수 있는 DFS 알고리즘을 참고하여 적절히 차용하였다.  

# Abort And Rollback

## Abort

Transaction이 abort 되는 경우는 크게 아래 두가지이다.

```text
1. find 또는 update 동작이 잘못되었을 때 (e.g. record가 존재하지 않음)
2. Deadlock이 감지되었을 때
```

이 때 해당 trx id를 가진 모든 lock_t 오브젝트를 release 해주고, Transaction Manager에서 해당 transaction에 해당되는 element 등을 삭제해준다. 이후에 Rollback을 실행한다.

이후에 같은 trx id로 들어오는 find, update는 모두 실패처리된다.

## Rollback

### Log

```c++
struct TrxLog {
    LogType log_type_;
    uint32_t table_;
    pagenum_t page_;
    int64_t key_;
    std::string before_;
    std::string after_;
}
```

로그는 위와 같은 멤버들을 가지는 struct로 구현하였다. find와 update가 각각 끝날 때면 해당 trx id에 작업 내용을 로그로 구성하여 추가해주는 방식으로 구현하였다.

### 되돌리기

Abort 시에 rollback을 진행하는데, 해당 transaction에 추가된 로그들을 역순으로 되돌리되, update에 대해서만 before_에 저장된 값으로 되돌려주는 작업을 하도록 하였다.

# Buffer Lock & Page Lock

Buffer Lock은 Buffer의 LRU 리스트에 수정이 가해지고 있다고 판단될 때 걸어주었고, Page Lock은 Page 가 사용중이라고 판단될 때 걸어주었다. 따라서 Eviction 시에 먼저 page lock을 획득하도록 구현하였는데, tail에서 lock을 획득할 때까지 기다리는 방식으로 구현하였으므로 insert 또는 delete 시에 버퍼 크기가 충분하지 않다면 해당 page lock을 획득하지 못해 무한히 기다리게 되는 부작용이 있을 수 있다.