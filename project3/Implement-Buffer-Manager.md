# 개요
## 환경
```
1. Ubuntu 18.04 (테스트)
2. Ubuntu 20.04 (실제 소스코드 작성)
3. g++ 7.5.0
4. C++ 17
```
## 컴파일 옵션
```
-g -fPIC -I -std=c++17 -W -Wall
```
## 소스코드
```
- enums.h
- file.cpp (file.h)
- page.cpp (page.h)
- file_manager.cpp (file_manager.h)
- buffer_manager.cpp (buffer.h, buffer_manager.h)
- bpt_node.cpp (bpt_node.h, bpt_node_impl.hpp)
- bpt.cpp (bpt.h)
- bptapi.cpp (bptapi.h)
- main.cpp
```
</br>

# 전체적인 구조
<center>
<img src="../uploads/project3/project3_overview.png" width="80%" height="80%">
</center> 
</br>

# FileManager & TableFile
## 구현에 대한 설명

1. project2 에서 file.c에 있던 함수들을 TableFile 클래스로 묶어 file descriptor와 같이 관리하였다. 
즉, file descriptor를 멤버 변수로 가지고, 기존 file.c의 함수들을 멤버 함수로 가지는 클래스인 TableFile을 정의하였다.

2. project2 에서 DBFileUtils 클래스의 이름을 FileManager로 변경하는 것이 적절하다고 판단하여 변경해주었다. 

3. 파일 Open, Close 등을 수행하며, TableFile 객체를 관리, 상위 레이어의 호출에 따라 
적절한 TableFile 객체를 찾아 해당 객체의 함수를 호출해주는 등의 역할을 한다. 

4. Table ID의 경우 현재 프로젝트에서 파일과 1 대 1 매칭이 되며, 
파일 경로와 TableFile 객체 Table ID를 matching하는 작업 등이 필요하므로 
FileManager 클래스에서 Table ID를 부여, 관리해주는 것이 합당하다고 판단하여 이와 같이 구현하였다.

</br>

## page_t 구조체
project2 에서 header page, free page는 page_t 구조체로, B+ Tree의 노드 역할을 하는 페이지는 node_t 구조체로 관리하여, 분리하였으나 project3 에서는 버퍼까지는 같은 형태로 관리해야 한다고 판단하여 page_t와 node_t를 union으로 묶어 page_t 구조체 하나로 관리하였다.

</br>

# BufferManager
## 구현에 대한 설명

1. BufferManager 클래스에서는 Buffer를 관리하는 여러 멤버 함수들을 구현하였다. LRU policy를 통한 Victim 선정, pin 관리 등을 해주는 클래스이다.

2. 버퍼는 BufferBlock 구조체 배열로 구현하였으며, 버퍼 초기화 시 파라미터로 받은 크기로 동적할당하였다. 

3. LRU policy 구현을 위한 linked list의 경우에는 next, prev block을 포인터가 아닌 배열의 인덱스로 가리키도록 구현하였다. 

4. 위와 같이 구현하면, 한 번 버퍼에 올라온 페이지는 eviction 되거나 table close, 또는 shutdown 등의 이유로 버퍼에서 내려갈 때까지 버퍼 배열의 같은 인덱스에 위치하므로 버퍼 블록의 위치를 찾기가 쉬워진다.

5. project2에서는 file.h의 API였던 file_alloc_page, file_free_page의 역할을 버퍼매니저에서 일부 담당하도록 하였다. 특히 file_free_page의 경우 delete_node라는 함수로, 기능 전체가 버퍼매니저에서 구현되었다.

6. STL의 unordered map을 활용하여 (table id, page number)을 키로, 해당하는 버퍼의 블록 인덱스를 value로 가지는 해시 테이블을 만들고, 후에 버퍼 블록을 찾을 때 이 해시 테이블을 사용하도록 하였다. 키가 pair 형태이므로 해시 함수는 STL의 해시 함수로 구한 두 해시 값을 XOR 하는 방식으로 구현하였다. 

    [[해시함수는 이곳을 참조하였다. | https://stackoverflow.com/questions/5889238/why-is-xor-the-default-way-to-combine-hashes]]

</br>

## BufferBlock 구조체
기본적으로 명세를 따랐으나, is_pinned 대신 pin_count를 사용하여, 페이지를 사용할 때 pin count를 올려주고, 페이지 사용을 끝낼 때 pin count를 내려주는 방식을 채택하였다.

</br>

# HeaderNode & BPTNode
## 구현에 대한 설명

1. page_t의 일종의 wrapper 역할을 하는 클래스로 구현하였다. 

1. pin과 is_dirty 관리의 용이성, 그리고 노드의 데이터 접근의 편의성을 위해 HeaderNode와 BPTNode 클래스를 구현하였다.

2. HeaderNode의 경우 Header page를 불러와, root page nuber을 읽거나 변경할 수 있도록 하였다.

3. BPTNode의 경우 템플릿을 통하여 leaf 노드와 internal 노드를 구분할 수 있게 하였으며, index layer의 함수들에서 이 두 노드를 구분하지 않고 사용할 때에는 어느 타입으로 선언해도 되도록 하였다. 

4. BPTNode의 경우 노드의 각종 데이터를 읽고 변경할 수 있도록 하였으며 read_data 함수는 child 혹은 record 배열 (leaf 노드냐,internal 노드냐에 따라) 의 const reference를, write_data 함수는 child 혹은 record 배열의 reference를 반환하도록 하였다. 또한, 값을 변경한 경우에는 is_changed 라는 변수를 true로 설정해주어 후에 버퍼 블록의 is_dirty에 반영될 수 있도록 하였다.

5. 버퍼에서 페이지 데이터 가져오기, pin count 관리, is dirty 관리 등이 모두 이들 클래스 내부에서 이루어지도록 하여 실제 index layer에서 크게 신경쓰지 않고 사용할 수 있도록 하였다.

</br>

# 기타
BPT 클래스나 bptapi의 경우 추가적인 함수들이 구현되거나 약간의 변경이 이루어졌다.

</br>

# 테스트
```python
import random

f = open("./test/test.txt", 'w')
f.write("z 5\n")
for i in range(1, 11):
    f.write("o ./t" + str(i) + ".db\n")
test_list_insert = list(range(1, 100001))
test_list_insert2 = list(range(200, 301))
test_list_delete = list(range(200, 99901))
test_list_find = list(range(1, 100001))
random.shuffle(test_list_insert)
random.shuffle(test_list_delete)

test_file_list = list(range(1, 11))
random.shuffle(test_file_list)

for j in test_file_list:
    for i in test_list_insert:
        f.write("i " + str(j) + " " + str(i) + " test " + str(i) + "\n")
    for i in test_list_delete:
        f.write("d " + str(j) + " " + str(i) + "\n")
    for i in test_list_find:
        f.write("f " + str(j) + " " + str(i) + "\n")
    for i in test_list_insert2:
        f.write("i " + str(j) + " " + str(i) + " test " + str(i) + "\n")

for j in test_file_list:
    f.write("p " + str(j) + "\n")
f.write("s")

f.close()
```
위와 같이 테스트 코드를 작성하여 테스트를 진행하였다.  
파일 10개를 열고, 각각 1부터 10만까지 삽입한 다음, 200에서 99900까지 삭제하고, 다시 200부터 300까지 삽입한 후 1부터 10만까지 탐색 후 트리를 출력하는 테스트 코드이다. 즉, 대략 300만번의 operation이 이루어지는 셈이다.   
z 5 는 DB를 버퍼크기 5로 초기화하겠다는 의미로, z 10000으로도 테스트해보았다.

<img src="../uploads/project3/test1.png" width="100%" height="100%">
</br>

위와 같이 트리에 잘 삽입과 삭제가 이루어졌음을 알 수 있다.  

버퍼 크기가 5였을 때의 실행 시간은 아래와 같았다.  
<img src="../uploads/project3/time1.png" width="30%" height="30%">
</br>

버퍼 크기가 10000이었을 때의 실행 시간은 아래와 같았다.  
<img src="../uploads/project3/time2.png" width="30%" height="30%">
</br>

버퍼의 크기가 크면 실행 시간이 감소하는 것을 확인할 수 있었다.