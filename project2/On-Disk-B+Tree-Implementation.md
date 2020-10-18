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
- file.cpp (file.h)
- page.cpp (page.h)
- fileutils.cpp (fileutils.h)
- bpt.cpp (bpt.h)
- bptapi.cpp (bptapi.h)
- main.cpp
```
</br>

# 전체적인 구조
<center>
<img src="../uploads/project2/milestone2/project_structure.png" width="50%" height="50%">
</center> 
</br>

- file.h에 명세된 요구된 함수를 포함해서 필요하다고 생각되는 함수와 page_t 구조체를 선언하고, file.cpp에 정의하였다. file.cpp에 정의된 함수들에서는 실제로 system call을 통하여 디스크에 읽고 쓰는 작업을 진행하도록 하였다.  
- page.h에는 인덱스 레이어에서 page_t 구조체를 바로 가져다 쓰는 것은 어려울 수도 있겠다고 판단하여 새롭게 node_t 구조체를 만들어주었다.  
- fileutils.h에는 DBFileUtils 클래스를 선언하고, fileutils.cpp에서 멤버 함수들을 정의해주었다. 해당 클래스에서는 상위 레이어의 요청에 따라 file.h의 함수들을 호출하도록 하였다.  
- bpt.h에는 BPT 클래스를 선언하고 bpt.cpp에서 멤버 함수들을 정의해주었다. 실제로 인덱스 레이어의 역할을 하는 클래스이다.  
- bptapi.h에는 명세에 요구된 4개의 함수를 선언하고, bptapi.cpp에서 정의하였다.
</br> </br>

# file.h & file.cpp
```C++
struct page_t {
    union page
    {
        uint8_t bytes[PAGE_BYTE];
        struct header_page
        {
            pagenum_t free_page_num;
            pagenum_t root_page_num;
            uint64_t number_of_pages;
        } header_page_;
        struct free_page
        {
            pagenum_t next_free_page_num;
        } free_page_;
    } page_;
};
```
page_t의 구조는 위와 같다. 페이지를 union으로 선언하여 header page나 free page 모두 page_t 하나로 관리할 수 있도록 하였다.
</br> </br>

```C++
pagenum_t file_alloc_page();

void file_free_page(pagenum_t pagenum);

void file_read_page(pagenum_t pagenum, page_t* dest);

void file_write_page(pagenum_t pagenum, const page_t* src);

int file_open_db(const char* pathname);

int file_close_db(int fd);

void set_current_db_fd(int fd);
```
위는 file.h에 선언된 함수들이다. 
file_alloc_page, file_free_page, file_read_page, file_write_page는 명세에서 요구한 함수들이고, 나머지는 필요하다고 판단된 함수들을 선언하였다.  
* **set_current_db_fd** 함수는 file.cpp에 정의되어있는 전역변수 current_db_fd에 인자로 받은 값을 저장하는 함수이다. file.h에 선언된 다른 함수들은 이 전역변수에 저장되어있는 file descriptor 값으로 파일을 읽고 쓰는 작업을 하게 된다.
* **file_open_db** 함수에서는 인자로 받은 파일명으로 open 시스템 콜을 통해 db 파일을 여는 함수이다. 이후에 반환받은 file descriptor 값을 set_current_db_fd 함수를 통해 current_db_fd 값에 저장해준다. 만약 열어준 파일이 크기가 0이라면, 즉 새로 생성한 파일이라면 헤더 페이지를 생성해준다.
* **file_close_db** 함수에서는 인자로 받은 file descriptor를 통해 close 시스템 콜로 파일을 닫는다.
* 이외에 file.cpp에 선언된 **file_write_page_array** 함수에서는 인자로 주어진 page_t 배열을 파일에 쓰고, **file_alloc_page**와 **file_wirte_page** 함수에서 이를 호출하는 방식으로 각각의 목적을 달성하게 된다.
* **file_alloc_page**의 경우 free page list에서 페이지를 가져오되, 헤더의 free page num이 0일 때, 즉 더 이상 free page가 없는 경우 FREE_PAGE_GROWTH (현재는 20으로 설정) 개수 만큼의 free page를 새로 만들어준다.

</br>

# page.h & page.cpp
실제로 index layer 단에서 쓸 수 있는 node_t 구조체를 정의하였다.
``` C++
struct node_t {
    struct header {
        pagenum_t parent;
        bool is_leaf;
        uint32_t number_of_keys;
        char reserved[104];
        pagenum_t spare_page;
    } header_;
    union body
    {
        std::array<record_t, 31> records;
        std::array<child_t, 248> childeren;

        body();
    } body_;
};
```
Internal node와 Leaf node가 Header은 동일하지만 Body는 다르기 떄문에 body의 경우 Union으로 선언해 Internal 노드는 children 멤버, Leaf 노드는 records 멤버를 사용하면 되도록 하였다.

# fileutils.h & fileutils.cpp
* fileutils.h에 선언된 **DBFileUtils** 클래스는 파일에 읽고 쓰는 작업을 관리하는 매니저 역할을 하는 클래스이다. 상위 레이어, 여기서는 인덱스 레이어에서 요구하는 경우 파일에서 페이지, 즉 노드를 읽어오거나 노드를 파일에 쓰는 작업을 하되, file.h의 함수들을 호출해서 해당 작업들을 하도록 하였다.  
* 이 클래스는 싱글턴 패턴을 활용하여 전체에 하나의 객체만 존재하도록 하였으며, 후에 여러 파일을 열어야 할 상황을 가정하여 여러 파일의 file descriptor 값을 넣어두어 관리할 수 있는 table_list 멤버를 가지도록 하였다.  
* 이외에 **bpt_node** 라는 새로운 구조체를 정의하여 생성자에서 자동으로 파일에서 읽어오도록 하는 등, 조금 더 쉽게 노드를 관리할 수 있도록 하였다.
</br></br> 

# bpt.h & bpt.cpp
* bpt.h에 선언된 **BPT** 클래스는 Index Layer 역할을 하는 클래스이다.
* 원래 bpt.c에 구현된 내용을 수정하는 방식으로 작성하였다.
* 기존 bpt.c에서는 Leaf 노드의 경우 pointer[order - 1]가 right sibling을 가리키도록 하였고, Internal 노드의 경우 pointer[0]가 left-most child를 가리키도록 되어있었는데, 여기서는 이 둘의 역할을 하는 부분이 따로 있었다. 해당 부분은 node_t를 정의하면서 spare_page라고 이름붙였다.
* 기존 bpt.c에서는 pointer와 key를 각각의 배열로 관리하였으나, 여기서는 (key, child), 또는 (key, value)로 묶어 하나의 배열로 관리하고 있으므로 이에 따르는 차이들도 반영해주었다.