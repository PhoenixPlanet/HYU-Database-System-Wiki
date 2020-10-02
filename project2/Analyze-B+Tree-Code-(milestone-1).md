# B+ 트리란?

![B+TREE](uploads/85f52febc23c9e2a7d97425339cc8256/Shape_of_bplustree.png) </br> </br>
B+ 트리의 특징을 간단히 정리해보자면 아래와 같다.
- 기본적으로 B-Tree와 모습이 유사한 Balanced Tree의 일종이다.  

- B-Tree와는 달리 Leaf 노드에 모든 record data가 정렬된 상태로 저장이 된다. </br> Leaf 노드를 제외한 노드는 Internal node로써, 각 키의 값은 right subtree의 첫 번째 키의 값으로 결정된다.
- B-Tree와 같이 B+ Tree의 각 노드는 최대 트리의 차수($order$)만큼의 자식을 가질 수 있고, $`k`$의 개의 자식을 가진 노드는 $`k-1`$개의 키 값을 가지게 된다.
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
<img src="uploads/71d6750feb241e6aef7d11f90c8bb353/delete_call_path.png" width="80%" height="80%">  

위 그림은 delete operation이 이루어질 때의 call path를 그림을 통해 나타낸 것이다. </br>
1. 먼저 삭제하고자 하는 값과, 해당 키를 가진 Leaf 노드를 탐색하여 없는 경우는 바로 끝낸다.

2. 탐색한 Leaf 노드에서 삭제하고자 하는 delete_entry 함수를 호출한다.  
아래는 delete_entry 함수 내부에서의 설명이다.
3. 키 값을 노드에서 삭제 한다.
4. 남은 key의 개수가 B+ 트리의 최소 조건을 만족하면 끝낸다.
5. 만약 남은 key의 개수가 최소 개수보다 적은 경우에는 Merge 등을 통해 적절히 트리를 변경한다. 
6. Merge 후에도 key의 가능한 최대 개수를 넘지 않는다면 coalesce_nodes 함수를 호출해 바로 노드를 Merge하고, 그렇지 않은 경우에는 redistribute_nodes 함수를 호출하여 키를 재분배하는 과정을 거친다. 이 두 함수의 자세한 내용과 call path에 대한 설명은 밑의 Merge 과정 설명에서 하도록 하겠다. </br></br>

# 2. Detail flow
