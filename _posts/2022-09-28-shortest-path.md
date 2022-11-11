---
title: Shortest Path Algorithms
author: Equinox134
date: 2022-09-28
categories: [Computer Science, Algorithm]
tags: [Computer Science, Algorithms, Graph Algorithm, C++]
math: true
excerpt: Explenation of algorithms for finding the shortest path
---
<!--more-->

## Introduction

The shortest path problem is(as the name suggests) a problem where you have to find the shortest path from one node to another in a graph. 
While there are many algorithms for finding the shortest path in a graph, here I want to introduce 4 of them; BFS, Dijkstra's algorithm, Bellman-Ford algorithm, 
and Floyd-Warshall algorithm.

## BFS

BFS is the (what I think) the simplest way to find the shortest path. However, it can only be used if the graph doesn't have any weights. 
There is a variation of the standard BFS known as 0-1 BFS, which can be used on graphs where the weights are either 0 or 1, but that's not explained here.

### Implementation

The following is an implementation of a function that returns the length of the shortest path in a given graph(-1 if unreachable).

```cpp
vector<int> graph[V+1]; //adjacency list of the graph, V is the number of nodes
int BFS(int start, int end){
  queue<int> q; bool visited[V+1]={};
  q.push(start); visited[start] = true;
  
  int len = 0; //the length of the traveled distance
  while(!q.empty()){
    int sz = q.size();
    for(int i=0; i<sz; i++){
      int cur = q.front(); q.pop();
      
      if(cur == end) return len;
      for(int next : graph[cur]){
        if(visited[next]) continue;
        q.push(next);
        visited[next] = true;
      }
    }
    len++;
  }
  return -1;
}
```

As you can see, the code isn't that different from a regular BFS code. The one difference is that the code above first gets the size of the queue, then 
runs the algorithm that many times. This way, we can search nodes that are an equal amount of distance away from the start, then increase the total distance.
Since the algorithm searches each node in order of increasing distance, the traveled distance when we first encounter the end node will automatically be the shortest
distance.

The time complexity of the algorithm is $O(V+E)$, because it's just a BFS search.

While BFS is simple and easy to implement, a lot of graphs have weights. Now lets look at algorithms that are used in weighted graphs.

## Dijkstra's Algorithm

Dijkstra's algorithm is an algorithm that can find the shortest path in a graph with weights. The only restraint is that all the weights must be positive.

Dijkstra's algorithm uses a greedy approach to find the shortest path. Initialy, the distance to all nodes are set to infinity, except for the start node, 
which is set to 0.

Then, the following process is repeated until the distance to every node has been determined; 
1. For every neighbouring node of the current node, we update the distance. More specifically, if the current distance to the next node is $C$, the distance to the 
current node $A$, and the weight of the edge between the two nodes $W$, the distance to the node is updated if $A+W<C$. This action is known as **relaxing**.
2. Mark the current node visited. This means we never change the distance of this node again.
3. Choose the neighbour nodes with the shortest distance, and make that node the current node.

### Implementation - 1

The following is an implementation of Dijkstra's algorithm.

```cpp
vector<pair<int,int> > graph[V+1]; //adjacency list of the graph, V is the number of nodes
int dist[V+1]; //an array that stores the minimum distance from a given node
int chk[V+1]; //an array that keeps track of whether a node distance has been chosen
void Dijkstra(int start){
  for(int i=1; i<=V; i++){ //reset dist and chk array
    dist[i] = 2e9;
    chk[i] = 0;
  }
  dist[start] = 0; chk[start] = 1;
  int now = start;
  
  //find distance for every node
  for(int i=0; i<V; i++){
    //check every neighbouring node
    for(int i : graph[now]){
      int nxt = i.first, cost = i.second;
      if(!chk[nxt]){
        if(dist[nxt] > dist[now] + cost)
          dist[nxt] = dist[now] + cost;
      }
    }
    
    //job done on node now
    chk[now] = 1;
    //find node with minimum distance that isn't done yet
    int min = 2e9;
    for(int i=1; i<=V; i++){
      if(!chk[i]){
        if(min > dist[i]){
          min = dist[i];
          now = i;
        }
      }
    }
  }
}
```

The code takes $O(V)$ time to find the closest node, and since this is repeated V times, the total time complexity would be $O(V^2)$.

### Using a Priority Queue

Rather than looping V times to find the closest node, we can make use of a priority queue ans find it in $O(logV)$ time.

The algorithm starts by reseting the distance to all nodes to infinity, except the start node, which is set to 0. Then, we add {0,start} to the priority queue, 
0 meaning the distance, start meaning the node. After that, the following is repeated until the priority queue becomes empty.
1. Pop the first element in the priority queue. The element should contain the node with the minimum distance, and the distance. Lets call them now and cost. 
If cost is larger than the current distance to now, we ignore/continue.
2. Otherwise, we relax the distance to all neighbouring nodes of now. If the distance is updated, we insert a pair {distance,node} into the priority queue.

### Implementation - 2

The following is an implementation of Dijkstra's algorithm using a priority queue. Note that the code uses a max heap(since the syntax is shorter), and so we multiply 
-1 to the distance whenever a new pair is being inserted. This way, the pair with a smaller distance would be at front.

```cpp
vector<pair<int,int> > graph[V+1]; //adjacency list of the graph, V is the number of nodes
int dist[V+1]; //an array that stores the minimum distance from a given node
void Dijkstra(int start){
  for(int i=1; i<=V; i++) dist[i] = 2e9; //reset distance
  
  priority_queue<pair<int,int> > pq;
  pq.push({0,start}); dist[start] = 0;
  
  while(!pq.empty()){
    int now = pq.top().second, -cost = pq.top().first; //multiply -1 whenever pushed, multiply -1 when we pop
    pq.pop();
    
    //if the distance is longer, ignore
    if(cost > dist[now]) continue;
    
    for(int i : graph[now]){
      int nxt = i.first, newCost = cost + i.second;
      if(newCost < dist[nxt]){
        dist[nxt] = newCost;
        pq.push({-newCost,nxt});
      }
    }
  }
}
```

If you use a min heap in while implementing the code above, there is no need to multiply -1.

The time complexity of this algorithm is $O(ElogV)$, where E is the number of edges, because a total of E edges are pushed into the priority queue, 
and it takes $O(logV)$ time to find the closest node.

## Bellman-Ford Algorithm

The Bellman-Ford algorithm, like Dijkstra's algorithm, is an algorithm that finds the shortest path from one node to all other nodes. Unlike Dijkstra's algorithm, 
however, the Bellman-Ford algorithm can be used on graphs that have negative weights, and can be used to find whether a graph contains a negative cycle.

Similar to Dijkstra's algorithm, the Bellman-Ford algorithm also proceeds by relaxation. The difference is, while Dijkstra's algorithm greedily selected the nodes to 
relax, the Bellman-Ford algorithm simply relaxes all edges, and repeats this $V-1$ times, where $V$ is the number of nodes.

### Implementation

The following is an implementation of the Bellman-Ford algorithm.

```cpp
vector<pair<int,int> > graph[V+1]; //adjacency list of the graph, V is the number of nodes
int dist[V+1]; //an array that stores the minimum distance from a given node
void BellmanFord(int start){
  for(int i=1; i<=V; i++) dist[i] = 2e9; //reset dist
  dist[start] = 0; //set distance to start as 0
  
  for(int i=0; i<V-1; i++){
    for(int j=0; j<V; j++){
      for(auto x : graph[j]){
        int w = x.second;
        int node = x.first;
        if(dist[node] > dist[j] + w) dist[node] = dist[j] + w;
      }
    }
  }
}
```

The implementation is pretty straight-forward, without much to explain. If you want to check for a negative cycle, all you have to do is repeat the relaxation process 
once more. During the relaxation process, if a distance is updated, this means that a negative cycle exists.

The time complexity of this algorithm is $O(VE)$, where E is the number of edges.

## Floyd-Warshall Algorithm

The Floyd-Warshall algorithm, unlike the previous algorithms, is an algorithm that finds the shortest path from every starting node to every end node. 
The way the Floyd-Warshall algorithm works is by looping through all pairs of start and end nodes(lets call them $S$ and $E$), then choosing a third node(lets call it $K$), then updating the distance from $S$ to $E$ if $dist(S,E) > dist(S,K) + dist(K,E)$(where $dist(A,B)$ is the distance from node $A$ to node $B$).

### Implementation

The following is an implementation of the Floyd-Warshall algorithm.

```cpp
int adj[V+1][V+1]; //adjacency matrix of the graph, where V is the number of nodes
//if i = j then adj[i][j] = 0, if there is an edge between i and j then adj[i][j] = w(the weight of the edge)
//otherwise adj[i][j] = INF
void FloydWarshall(){
  for(int k=0; k<V; i++){
    for(int i=0; i<V; j++){
      for(int j=0; j<V; k++){
        if(adj[i][j] > adj[i][k] + adj[k][j])
          adj[i][j] = adj[i][k] + adj[k][j];
      }
    }
  }
}
```

One thing to note in the implementation is that the loops in order of passing point -> starting point -> endpoint.

The time complexity of the code is $O(V^3)$.

## Conclusion

So those were the algorithms I wanted to talk about here. There are more, but those should be enough to solve most problems related to shortest paths.
