---
title: DFS总结2
date: 2020-01-10 14:40:09
tag: 
    - "leetcode"
    - "algorithm"
categories:
    - "technique"
---

## DFS 大全2

在此写一个dfs的相关题目总结。

### 目录

<font color=green>**Easy**</font> 
<font color=orange>**Medium**</font> 
<font color=red>**Hard**</font>

Problem | Notes | 自评
------------ | ------------- |-------
<a href="#694">**694**</a> <font color=orange>**Medium**</font> Number of Distinct Islands| Typical DFS but could be tricly! | <font color=orange>**Medium**</font> Very smart and think about it carefully.
<a href="#126">**126**</a> <font color=red>**Hard**</font> word ladder II| New BFS problems, the element in queue in a path! | <font color=red>**Hard**</font> It is easy to find shortest path with BFS, but how about finding multiple paths?
<a href="#351">**351**</a> <font color=orange>**Medium**</font> Android Unlock Patterns| A very interesting problem but not so hard. | <font color=red>**Medium**</font> Some solutions are symmetric so you do not have to do DFS on every entry.
<a href="#139">**139**</a> <font color=orange>**Medium**</font> Word Break| In fact it is a typical DFS problem with memo. | <font color=orange>**Medium**</font> You have to deeply REMEMBER this solution.
<a href="#140">**140**</a> <font color=red>**Hard**</font> Word Break II | This is the follow-up from the last one.  | <font color=orange>**Medium**</font> This is still dfs+memo. You have to know how to use memo in this case!
<a href="#505">**505**</a> <font color=orange>**Medium**</font> The Maze II | You can do it with a standard DFS or BFS. But either solution may visit a node more than once. This is a typical shortest path solution, and either BFS or DFS is an instance of the generic solution to SP. The BFS solution would be modified into Dijkstra's algorithm. | <font color=orange>**Medium**</font> It is not a normally DFS/BFS solution as one node might be visited for more than once.




### <a id="694">694 Number of Distinct Islands</a>

Given a non-empty 2D array grid of 0's and 1's, an island is a group of 1's (representing land) connected 4-directionally (horizontal or vertical.) You may assume all four edges of the grid are surrounded by water.

Count the number of distinct islands. An island is considered to be the same as another if and only if one island can be translated (and not rotated or reflected) to equal the other.

**Example**:

    11011
    10000
    00001
    11011

The output is 2.

这个题目不难，典型的dfs，先看一个解法。

```java
class Solution {
    public int numDistinctIslands(int[][] grid) {
        // we could still use dfs to solve this problem
        // we may record the "direction" of dfs to identify the shape of an island
        Set<String> set = new HashSet<>();
        for (int i = 0; i < grid.length; i++) {
            for (int j = 0; j < grid[0].length; j++) {
                if (grid[i][j] == 0) continue;
                
                StringBuilder sb = new StringBuilder(); // origin
                dfs(i, j, grid, sb, 'o'); // origin
                set.add(sb.toString());
                System.out.println(sb);
                
            }
        }
        return set.size();
    }
    
    private void dfs(int i, int j, int[][] grid, StringBuilder sb, char direction) {
        if (i < 0 || j < 0 || i >= grid.length || j >= grid[0].length || grid[i][j] == 0) return;
        
        grid[i][j] = 0;
        sb.append(direction);
        
        dfs(i-1, j, grid, sb, 'u');
        dfs(i+1, j, grid, sb, 'd');
        dfs(i, j-1, grid, sb, 'l');
        dfs(i, j+1, grid, sb, 'r');
        
        sb.append('b'); // This is amazing!!!!!
        
    }
}
```

为什么要在dfs做完之后加一个"b"呢？假如没有的话，请考虑一下这种情况：

     1 1 0
     0 1 1
     0 0 0
     1 1 1
     0 1 0

这次做完两个大的dfs之后，会发现两个字符串均为`ordr`，也就是说这种方法还是无法区分某些不同的island。在后面在一个`b`等于mark了一个dfs的结束，这样，两个island要完全相同，才能保证这两个对应的字符串是一样的。

### <a id="126"> 126 Word Ladder II </a>

Given two words (beginWord and endWord), and a dictionary's word list, find all shortest transformation sequence(s) from beginWord to endWord, such that:

- Only one letter can be changed at a time
- Each transformed word must exist in the word list. Note that beginWord is not a transformed word.

In BFS, it is easy to find the shortest path, but how can you find all possible shortest paths?

There are several points you need to pay attention to:
- How can you save the paths? One good way is to use the path as the element of the queue;
- How can you make sure that some nodes could be visited twice? The solution is here is really interesting: **we only mark the nodes as visited only after we have done this level**. This can ensure two different paths can both visit this node if they reach it at the same level. It also make sure this node is not visited by the higher level;
- `minLevel` is used to avoid unnecessary BFS.

```cpp
class Solution {
public:
    vector<vector<string>> res;
    int level = 1;
    int minLevel = INT_MAX;
    
    vector<vector<string>> findLadders(string beginWord, string endWord, vector<string>& wordList) {
        // just like finding all possible paths?
        // maybe bfs?
        unordered_map<string, vector<int>> map;
        map[beginWord] = findNeighbor(beginWord, wordList);
        for (string& word: wordList) map[word] = findNeighbor(word, wordList);
        
        // I think one problem with BFS is
        // how can you make sure that some elements can be visited more than once?
        //basic idea, we mark the node is visited obly after the level is done
        // thus, two different paths which reaching a node at same level can both
        // visit this node
        queue<vector<string>> q;
        q.push({beginWord});
        unordered_set<string> wordSet;
        unordered_set<string> visited;
        for (string& str: wordList) wordSet.insert(str);
        
        while (q.size() > 0) {
            vector<string> curPath = q.front();
            q.pop();
            
            if (curPath.size() > level) {
                // enter new level
                if (curPath.size() > minLevel) {
                    break;    
                } else {
                    level = curPath.size();
                }
                // add visited node
                for (string str: visited) wordSet.erase(str);
                visited.clear();
            }
            
            string lastWord = curPath.back();
            for(int i: map[lastWord]) {
                string& newWord = wordList[i];
                
                // is visited?
                if (wordSet.find(newWord) == wordSet.end()) continue;
                
                visited.insert(newWord);
                // prepare a new path
                vector<string> path = curPath;
                path.push_back(newWord);
                if (newWord == endWord) {
                    res.push_back(path);
                    minLevel = level;
                } else {
                    q.push(path);
                }
            }
        }
        return res;
    }
    
    
    vector<int> findNeighbor(string& word, vector<string>& words) {
        // given a word, find neighbor
        vector<int> vec;
        for (int i = 0; i < words.size(); i++) {
            string& s = words[i];
            if (s.size() != word.size()) continue;
            int diff = 0;
            for (int j = 0; j < word.size() ; j++) {
                if (s[j] != word[j]) diff ++;
                if (diff > 1) break;
            }
            if (diff == 1) vec.push_back(i);
        }
        return vec;
    }
};
```

Share another java solution: we may firstly use BFS solution to get the distance of all nodes, and then use DFS to get the results.

```java
class Solution {
    
    // bfs + dfs
    // bfs is responsible for find nodes and distance by level traversal
    // dfs is to find all results
    
    List<List<String>> res;
    
    public List<List<String>> findLadders(String beginWord, String endWord, List<String> wordList) {
        res = new LinkedList<>();
        Set<String> set = new HashSet<>(wordList);
        Map<String, Integer> distance = new HashMap<>();
        Map<String, Set<String>> neighbors = new HashMap<>();
        bfs(beginWord, endWord, set, distance, neighbors);
        // System.out.println(neighbors);
        // System.out.println(distance);
        List<String> list = new LinkedList<>();
        list.add(beginWord);
        dfs(beginWord, endWord, distance, neighbors, list);
        return res;
    }
    
    private void bfs(String startWord, String endWord, Set<String> allWords, Map<String, Integer> distance, 
                     Map<String, Set<String>> neighbors) {
        // assume we have put start word in the start set
        distance.put(startWord, 0);
        int level = 0;
        Queue<String> queue = new LinkedList<>();
        queue.offer(startWord);
        while (queue.size() > 0) {
            int size = queue.size();
            boolean shouldEnd = false;
            for (int i = 0; i < size; i ++) {
                String cur = queue.poll();
                List<String> nexts = getNeighbors(cur, allWords);
                neighbors.put(cur, new HashSet<>());
                for (String next: nexts) {
                    neighbors.get(cur).add(next);
                    if (!distance.containsKey(next)) {
                        distance.put(next, level + 1);
                        if (next.equals(endWord)) {
                            shouldEnd = true;
                        } else {
                            queue.offer(next);
                        }
                    }
                }
            }
            if (shouldEnd) break;
            level ++;
        }
    }
    
    private void dfs(String startWord, String endWord, Map<String, Integer> distance, 
                     Map<String, Set<String>> neighbors, List<String> curList) {
        
        if (startWord.equals(endWord)) {
            res.add(new LinkedList<>(curList));
            return ;
        }
        
        Set<String> nexts = neighbors.get(startWord);
        if (nexts == null) return ;
        for (String next: nexts) {
            if (distance.get(next) != distance.get(startWord) +  1) continue;
            curList.add(next);
            dfs(next, endWord, distance, neighbors, curList);
            curList.remove(curList.size() - 1);
        }
    }
    
    private ArrayList<String> getNeighbors(String node, Set<String> dict) {
        ArrayList<String> res = new ArrayList<String>();
        char chs[] = node.toCharArray();
        
        for (char ch ='a'; ch <= 'z'; ch++) {
            for (int i = 0; i < chs.length; i++) {
                if (chs[i] == ch) continue;
                char old_ch = chs[i];
                chs[i] = ch;
                if (dict.contains(String.valueOf(chs))) {
                    res.add(String.valueOf(chs));
                }
                chs[i] = old_ch;
            }
        }
        return res;
    }
}
```

### <a id="315"> 315 Android Unlock Patterns </a>

Given an Android 3x3 key lock screen and two integers m and n, where 1 ≤ m ≤ n ≤ 9, count the total number of unlock patterns of the Android lock screen, which consist of minimum of m keys and maximum n keys.

This is a very interesting problem, and guess what, it must come form Google! So this problem is not hard to solve. As for the work flow and logic, you can use DFS and back-tracing; the hard part is to find all possible next integer for a specific integer. But it is still trivial, as there is only 9 grid in total. Also note that we do not have to do dfs for every entry in the 2D array as the array is symmetric.

```java
class Solution {
    
    public int numberOfPatterns(int m, int n) {
        // OMG this question in sl interesting!
        // the basic work flow: recursion (backtracing)
        // how can you determine if two number can be connected?
        boolean visited[][] = new boolean[3][3];
        // for (int i = 0; i < 3; i++) {
        //     for (int j = 0; j < 3; j++) {
        //         dfs(i, j, m, n, visited);
        //         // System.out.println();
        //     }
        // }
        int ans = 0;
        ans += 4 * dfs(0, 0, m, n, visited);
        ans += 4 * dfs(0, 1, m, n, visited);
        ans += dfs(1, 1, m, n, visited);
        return ans;
    }
    
    private int dfs(int i, int j, int min, int max, boolean[][] visited) {
        // if (i < 0 || i>= 3 || j < 0 || j >= 3) return ;
        
        int res = 0;
        int new_min = Math.max(0, min - 1);
        int new_max = max - 1;
        
        if (new_min == 0) {
            res ++;
        }
        if (new_max == 0) {
            return res;
        }
        
        visited[i][j] = true;
        
        for (int x = 0; x < 3; x++) {
            for (int y = 0; y < 3; y++) {
                if (visited[x][y]) continue;
                int[] middle = getMiddle(i, j, x, y);
                if (middle == null || visited[middle[0]][middle[1]]) {
                    res += dfs(x, y, new_min, new_max, visited);
                }
                
            }
        }
        visited[i][j] = false;
        return res;
    }
    
    private int[] getMiddle(int x0, int y0, int x1, int y1) {
        int sum = x0 + y0 + x1 + y1;
        if (sum % 2 == 1) return null;
        int diffx = Math.abs(x0-x1), diffy = Math.abs(y0-y1);
        if (diffx == 1 && diffy == 1) return null;
        return new int[]{(x1+x0)/2, (y0+y1)/2};
    }
}
```

### <a id="139"> 139 Word Break </a>

Given a non-empty string s and a dictionary wordDict containing a list of non-empty words, determine if s can be segmented into a space-separated sequence of one or more dictionary words.

Note:
- The same word in the dictionary may be reused multiple times in the segmentation.
- You may assume the dictionary does not contain duplicate words.

This is a typical DFS + memo problem. It is not hard to write down, but it is very basic.

```java
class Solution {
    public boolean wordBreak(String s, List<String> wordDict) {
        // the recursion situation is like O(n!)
        // use dp
        // Time complexity: O(s.length() * wordDict.size()) in worst case
        int[] dp = new int[s.length()]; // dp[i] => substring(i) 
        Arrays.fill(dp, -1);
        return visit(s, 0, wordDict, dp);
    }
    
    private boolean visit(String s, int index, List<String> words, int[] dp) {
        if (index >= s.length()) return true;
        if (dp[index] == 1) return true;
        else if (dp[index] == 0) return false;
        
        // if dp[index] == -1 -> not visited
        for (String word: words) {
            if (s.startsWith(word, index)) {
                int nextIndex = index + word.length();
                if (visit(s, nextIndex, words, dp)) dp[index] = 1;
            }
        }
        
        if (dp[index] == -1) dp[index] = 0;
        return dp[index] == 1;
    }
}
```

In fact, we do need to make it too complicated. A simple pass through DP is enough.

```java
class Solution {
    public boolean wordBreak(String s, List<String> wordDict) {
        Set<String> set = new HashSet<>(wordDict);
        boolean[] dp = new boolean[s.length() + 1];
        
        dp[0] = true;
        for (int i = 1; i <= s.length(); i++) {
            for (int j = 0; j < i; j++) {
                if (!dp[j]) continue;
                if (set.contains(s.substring(j, i))) {
                    dp[i] = true;
                    break;
                }
            }
        }
        return dp[s.length()];
    }
}
```

The code is self-explanatory.

### <a id="140"> 140 Word Break II </a>

Given a non-empty string s and a dictionary wordDict containing a list of non-empty words, add spaces in s to construct a sentence where each word is a valid dictionary word. Return all such possible sentences.

In this case, the common solution is still DFS. But how can you save the results? Use the map.

```java
class Solution {
    public List<String> wordBreak(String s, List<String> wordDict) {
        Map<String, List<String>> map = new HashMap<>();
        return visit(s, wordDict, map);
    }
    
    public List<String> visit(String s, List<String> wordDict, Map<String, List<String>> map) {
        
        List<String> list = new LinkedList<>();
        
        if (s.length() == 0) {
            list.add("");
            return list;
        }
        
        if (map.containsKey(s)) return map.get(s);
        
        for (String str: wordDict) {
            if (s.startsWith(str)) {
                
                List<String> tmp = visit(s.substring(str.length()), wordDict, map);
                for (String suffix: tmp) {
                    list.add(str + (suffix.length() > 0? " ": "") + suffix);
                }
            }
        }
        map.put(s, list);
        return list;
    }
}
```

### <a id="505"> 505 The Maze II </a>

There is a ball in a maze with empty spaces and walls. The ball can go through empty spaces by rolling up, down, left or right, but it won't stop rolling until hitting a wall. When the ball stops, it could choose the next direction.

Given the ball's start position, the destination and the maze, find the shortest distance for the ball to stop at the destination. The distance is defined by the number of empty spaces traveled by the ball from the start position (excluded) to the destination (included). If the ball cannot stop at the destination, return -1.

The maze is represented by a binary 2D array. 1 means the wall and 0 means the empty space. You may assume that the borders of the maze are all walls. The start and destination coordinates are represented by row and column indexes.

This is a typical shortest path problem. For this problem, we have a generic solution:

Define "relax" an edge in the following code:

```java
private void relax(EdgeWeightedDigraph G, int v) {
    for (DirectedEdge e: G.adj(v)) {
         int w = e.to(); // w is another vertex connected by e
         if (distTo[w] > distTo[v] + e.weight()) {
            distTo[w] = distTo[v] + e.weight();
            edgeTo[w] = e;
        }
    }
}
```

Then we have a generic solution for shortest path: Initialize `distTo[s]` (`s` is the source/start of the graph) to 0 and all other `distTo[]` values to infinity, and proceed as follows: Relax any edge in this graph, continuing until no edges is eligible. For all vertices `w` reachable from `s`, the value of `distTo[w]` after this computation is the length of a shortest path from `s` to `w`.

We now have two solutions, DFS and BFS.

```java
class Solution2 {

    int[][] m;
    int[] s;
    int[] d;
    public int shortestDistance(int[][] maze, int[] start, int[] destination) {
        // dfs + memo
        m = maze;
        s = start;
        d = destination;
        int row = m.length, col = m[0].length;
        int[][] dp = new int[row][col];
        dp[s[0]][s[1]] = 1; // 0 is unvisited
        dfs(s[0], s[1], dp);
        return dp[d[0]][d[1]] - 1;
    }
    
    private void dfs(int i, int j, int[][] dp) {
        // we start from the start end, and visit all possible nodes in the current call
        // then update the dp matrix. We always choose the min distance
        if (i == d[0] && j == d[1]) return ;
        int[] dirs = {0, -1, 0, 1, 0};
        for (int p = 0; p < 4; p++) {
            int diffx = dirs[p], diffy = dirs[p+1];
            int x = i, y = j;
            int len = 0;
            while (x + diffx >= 0 && x + diffx < m.length && y + diffy >= 0 && y + diffy < m[0].length 
                   && m[x+diffx][y+diffy] == 0) {
                x += diffx; y += diffy;
                len ++;
            }
            int newLength = dp[i][j] + len;
            if (dp[x][y] > 0 && newLength >= dp[x][y]) continue;
            dp[x][y] = newLength;
            dfs(x, y, dp);
        }
    }
}
```

```java
class Solution {
    public int shortestDistance(int[][] maze, int[] start, int[] destination) {
        // we use the bfs to try!
        // If a node is updated, then put it in the queue again
        // This soluton could be modified to Dijkstra's algorithm!
        // we still have to use a distance matrix 2D
        int[][] distance = new int[maze.length][maze[0].length];
        for (int[] d: distance) Arrays.fill(d, Integer.MAX_VALUE);
        Queue<int[]> queue = new LinkedList<>();
        queue.offer(new int[]{start[0], start[1], 0});
        while (queue.size() > 0) {
            int[] cur = queue.poll();
            int i = cur[0], j = cur[1];
            int[] dirs = {0, -1, 0, 1, 0};
            for (int p = 0; p < 4; p++) {
                int diffx = dirs[p], diffy = dirs[p+1];
                int x = i, y = j;
                int len = cur[2];
                while (x + diffx >= 0 && x + diffx < maze.length && y + diffy >= 0 && y + diffy < maze[0].length 
                       && maze[x+diffx][y+diffy] == 0) {
                    x += diffx; y += diffy;
                    len ++;
                }
                
                // This is a way to reduce extra operation
                if (len > distance[destination[0]][destination[1]]) continue;
                
                if (distance[x][y] != 0 && distance[x][y] > len) {
                    distance[x][y] = len;
                    if (x != destination[0] || y != destination[1]) queue.offer(new int[]{x, y, len});
                }
            }
            
        }
        int res = distance[destination[0]][destination[1]];
        return res == Integer.MAX_VALUE? -1: res;
        
    }
}
```

Either solution is an instance of the generic solution to SP problem. BFS could be modified into Dijkstra's algorithm then.