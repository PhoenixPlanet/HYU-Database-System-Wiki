# B+ 트리란?

![B+TREE](uploads/85f52febc23c9e2a7d97425339cc8256/Shape_of_bplustree.png) </br> </br>

B+ 트리의 특징을 간단히 정리해보자면 아래와 같다.
- 기본적으로 B-Tree와 모습이 유사한 Balanced Tree의 일종이다.  

- B-Tree와는 달리 Leaf 노드에 모든 record data가 정렬된 상태로 저장이 된다. </br> Leaf 노드를 제외한 노드는 Internal node로써, 각 키의 값은 해당 키의 right subtree의 Leaf node의 첫 번째 키의 값으로 결정된다.
- B-Tree와 같이 B+ Tree의 각 노드는 최대 트리의 차수($`order`$)만큼의 자식을 가질 수 있고, $`k`$의 개의 자식을 가진 노드는 $`k-1`$개의 키 값을 가지게 된다.
- B+ Tree의 각 노드는 최소한 $`\left \lceil{order / 2}\right \rceil - 1`$개의 키를 가져야 한다.
- 삽입과 삭제 시에 Balancing 등 B+ Tree의 특징을 유지하기 위해 적절한 Merge와 Split이 이루어지게 된다.
- B+ Tree 와는 달리 record data가 모두 Leaf 노드에 저장이 되므로, 삽입과 삭제 역시 Leaf 노드에서 이루어지게 된다. </br></br>

# 1. Call Path
## A. insert
<img src="uploads/09f771815d3258cd084d05958b7221b8/insert_call_path.png" width="80%" height="80%">  

위 그림은 insert operation이 이루어질 때의 call path를 그림을 통해 나타낸 것이다. </br>
1. 먼저 중복을 막기 위해서 삽입하고자 하는 값이 이미 트리에 있는지 확인 후, 없다면 삽입을 시작한다. 

2. 만약 트리가 없다면, 즉 root가 null이라면 새로 루트 노드를 생성해 트리를 생성하고, 해당 값을 넣어준다. 
3. 트리가 있는 경우에는, 새로운 값을 집어넣을 Leaf 노드를 탐색한다. 
4. 찾은 Leaf 노드에 자리가 충분하다면, 즉 키 값의 개수가 $`order - 1`$ 보다 작다면 그대로 삽입한다.
5. 자리가 부족하다면 Split 후 삽입을 진행한다. 이 때 호출되는 함수 insert_into_leaf_after_splitting 의 자세한 내용과 call path에 대한 설명은 밑의 Split 과정 설명에서 하도록 하겠다. </br> </br>

## B. delete
<img src="uploads/62029432493cab5695806b71ad1389fa/delete.png" width="100%" height="100%">  

위 그림은 delete operation이 이루어질 때의 call path를 그림을 통해 나타낸 것이다. </br>
1. 먼저 삭제하고자 하는 값과, 해당 키를 가진 Leaf 노드를 탐색하여 없는 경우는 바로 끝낸다.

2. 탐색한 Leaf 노드에서 삭제하고자 하는 delete_entry 함수를 호출한다.  
아래는 delete_entry 함수 내부에서의 설명이다.
3. 키 값을 노드에서 삭제 한다.
4. 남은 key의 개수가 B+ 트리의 최소 조건을 만족하면 끝낸다.
5. 만약 남은 key의 개수가 최소 개수보다 적은 경우에는 Merge 등을 통해 적절히 트리를 변경한다. 
6. Merge 후에도 key의 가능한 최대 개수를 넘지 않는다면 coalesce_nodes 함수를 호출해 바로 노드를 Merge하고, 그렇지 않은 경우에는 redistribute_nodes 함수를 호출하여 키를 재분배하는 과정을 거친다. 이 두 함수의 자세한 내용과 call path에 대한 설명은 밑의 Merge와 Redistribute 과정 설명에서 하도록 하겠다. </br></br>

# 2. Detail Flow
## A. Split
<img src="uploads/ae831f69c4bf519d862d1eb75fb22cfd/insert_after_split.png" width="70%" height="70%"> </br>

위 그림은 Leaf node를 split 후 키를 insert하는 함수인 insert_into_leaf_after_splitting 의 call path를 나타낸 그림이다. </br>
함수별로 살펴보면서 Split 하는 과정을 자세히 알아보자.
- insert_into_leaf_after_splitting 함수  
<img src="uploads/8c1cbbaa5512b4b19729f5bc5e505ce6/Leaf_split.png" width="50%" height="50%"> </br>

insert_into_leaf_after_splitting 함수에서는 위 그림($`order = 4`$인 경우)과 같이 원래의 키들과 새로 삽입할 키를 temp_keys와 temp_pointers에 일단 위치에 맞게 집어넣은 후, leaf 노드를 쪼개게 된다. </br>
이후에 parent에 이를 반영해주기 위해 insert_into_parent 함수를 호출해주게 된다.
  
- insert_into_parent 함수
</br>insert_into_parent 함수에서는 크게 세 가지 경우로 나뉘어서 작동한다.
    ```
    1. 부모가 없는 경우: insert_into_new_root
    2. 부모 노드에 자리가 있는 경우 (부모가 가진 key 개수가 가능한 최대 개수보다 적음): insert_into_node
    3. 부모 노드에 자리가 없는 경우: insert_into_node_after_splitting
    ```

- insert_into_new_root 함수
</br>새로운 root 노드를 생성한다.

- insert_into_node 함수
</br>그냥 적절한 위치에 삽입해준다.

- insert_into_node_after_splitting 함수
</br>사실상 insert_into_leaf_after_splitting과 거의 비슷하게 작동한다. 이 함수에서 다시 insert_into_parent 함수를 호출하여 재귀적으로 부모를 타고 올라가며 insert를 진행하고, insert_into_new_root 나 insert_into_node가 호출될 때까지 반복한다.

## B. Merge and Redistribute
### 가. Merge (coalesce_nodes)
Merge 후에도 노드의 키 개수가 key의 가능한 최대 개수를 넘지 않는다면 coalesce_nodes 함수를 통해 Merge한다.</br>
1. 기본적으로 왼쪽 노드와 Merge한다. 만약 제일 왼쪽 노드라면, 오른쪽 노드와 Merge한다.

2. 왼쪽 노드에 키들을 이어 붙이되, Internal node의 경우 왼쪽 노드에 부모 노드에서 값을 가져와 붙인 후, key가 삭제된 노드의 값을 이어 붙인다. 
3. 부모 노드에서 원래 노드, 즉 키가 삭제된 노드를 가리키던 키를 삭제한다. 이 과정에서 재귀적으로 delete_entry 함수가 호출되며 거슬러 올라간다.

### 나. Redistribute (redistribute_nodes)
Merge 후에도 노드의 키 개수가 key의 가능한 최대 개수를 넘는다면 redistribute_nodes 함수를 통해 키를 재배열한다. </br>
1. 왼쪽 노드의 가장 오른쪽 키 값을 가져온다.

2. 만약 제일 왼쪽 노드라면 오른쪽 노드의 가장 왼쪽 키 값을 가져온다.
3. 이에 맞추어 parent에 반영해준다.