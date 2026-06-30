---
tags:
  - dsa
  - graph
---
# Depth first search
for any given graph be it directed or undirected the depth first search has the same effect
though it's called depth first **search**, it depends on the implementation whether the function does traversal or searching, it uses a LIFO ds to push the neighbors to and traverse them. 

## Example implementation of DFS using stack in Typescript 
- the weighted counter part would have another block of code that get's the neighbors sorted based on the weight so that they get visited first
- the below are 2 implementation of the same algo but with different underlying ds mechanisms used.

1. using the call stack for traversal for unweighted 
```javascript

const dfs = (graph: UnweightedGraph, startNode: NodeId, visited = new Set<NodeId>()) => {
  if (visited.has(startNode)) return;
  visited.add(startNode);
  for (const neighbor of graph[startNode]) {
    dfs(graph, neighbor, visited);
  }
};
```

2. using a separate stack ds for traversal
```javascript
const dfs = (graph: UnweightedGraph, startNode: NodeId, visited = new Set<NodeId>()) => {
  if (visited.has(startNode)) return;
  visited.add(startNode);
  const stack = [];
  stack.push(startNode);
  while (stack.length > 0) {
	  const current = stack.pop();
	  for (let neighbor of graph[current]) {
		  stack.push(neighbor);
	  }
  }
};
```


# Breadth first Search

this is the same concept as dfs but instead of pushing the neighbors on to a stack, you push them to a queue so that FIFO. 
this way the the traversal always goes layer by layer instead of nesting deep in the graph following one path till the end.


```javascript
const bfs = (graph: UnweightedGraph, startNode: NodeId, visited = new Set<NodeId>) => {
	const queue = [];
	queue.push(startNode);
	while (queue.length > 0) {
		const current = queue.shift();
		visited.add(current);
		for(let neighbor of graph[current]) {
			queue.push(neighbor)
		}
	}	
}
```