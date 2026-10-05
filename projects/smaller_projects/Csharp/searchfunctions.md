---
layout: default
title: Portfolio |  Dijkstra / Bi-directional BFS
---

[Home](../../../../index.md) / [Smaller Projects](../index.md) /

# Search Functions
This code is extracted from an assignment, including bi-directional breadth first search and dijkstra search algorithms.
```csharp
public List<string> BidirectionalBreadthFirstSearch(string start, string goal, Func<string, bool>? constraint = null){
    if(start == goal) return [];

    Queue<string> q1 = new();
    Queue<string> q2 = new();

    Dictionary<string, string?> parent1 = [];
    Dictionary<string, string?> parent2 = [];

    q1.Enqueue(start);
    parent1[start] = null;

    q2.Enqueue(goal);
    parent2[goal] = null;

    string? meet = null;

    bool Expand(Queue<string> q,Dictionary<string, string?> current,Dictionary<string, string?> other,ref string? meet,Func<string, bool>? constraint){
        if(q.Count == 0) return false;
        string node = q.Dequeue();
        foreach(RTBExit exit in map.Locations[node].Exits){
            string edge = exit.Name;
            string dest = map.Destination[(node, edge)];
            if(current.ContainsKey(dest)) continue;
            if(constraint != null && !constraint(dest)) continue;
            current[dest] = node;
            if(other.ContainsKey(dest)){
                meet = dest;
                return true;
            }
            q.Enqueue(dest);
        }
        return false;
    }

    while (q1.Count > 0 && q2.Count > 0){
        if(q1.Count <= q2.Count){
            if (Expand(q1, parent1, parent2, ref meet, constraint))
                break;
        }
        else{
            if (Expand(q2, parent2, parent1, ref meet, constraint))
                break;
        }
    }
    if (meet == null) return [];

    var nodes = new List<string>();
    string? cur = meet;
    while(cur != null){
        nodes.Add(cur);
        cur = parent1[cur];
    }
    nodes.Reverse();
    cur = parent2[meet];
    while(cur != null){
        nodes.Add(cur);
        cur = parent2[cur];
    }

    List<string> path = [];
    for(int i = 0; i < nodes.Count - 1; i++){
        string from = nodes[i];
        string to = nodes[i + 1];
        // find the exit that goes from `from` to `to`
        string? exitName = null;
        if(map.ExitLookup.ContainsKey((from,to)))
            exitName = map.ExitLookup[(from, to)];
        if (exitName == null) return [];
        path.Add(exitName);
    }

    return path;
}

public List<string> Dijkstra(string from, string to, Func<string, int> edgeCost, Func<string,string, int> heuristic, Func<string, bool>? constraint = null){
    PriorityQueue<string, (int cost, long tiebreaker)> queue = new();
    var parent = new Dictionary<string, (string? prev, string? exitUsed)>();
    
    long tiebreaker = 0;
    queue.Enqueue(from, (0,tiebreaker++));
    parent[from] = (null, null);

    while(queue.Count > 0){
        var node = queue.Dequeue();
        if(node == to) break;

        foreach(RTBExit exit in map.Locations[node].Exits){
            string edge = exit.Name;
            if(!map.Destination.ContainsKey((node, edge))) continue;
            string dest = map.Destination[(node, edge)];

            if(parent.ContainsKey(dest)) continue;

            // apply an optional constrain, like for example skip all nodes not in the same room
            if(constraint != null && !constraint(dest)) continue;

            parent[dest] = (node, edge);
            int h = heuristic(edge, dest);
            int c = edgeCost(edge);
            queue.Enqueue(dest, (c+h, tiebreaker++));
        }
    }

    // reconstruct path
    var path = new List<string>();
    string? cur = to;
    while(cur != from){
        if(cur == null) continue;
        var (prev, exitUsed) = parent[cur];
        if(exitUsed == null) continue;
        path.Add(exitUsed);
        cur = prev;
    }

    path.Reverse();
    return path;
}
```