# Frequently Asked DSA Questions — Top Product Companies

---

## Interview Framework — How to Solve Any DSA Problem

Always follow this structure. Interviewers evaluate process, not just the answer.

### Step 1: Understand the Problem (2-3 min)
- Read the problem carefully, identify input/output
- Ask clarifying questions:
  - Can input be empty or null?
  - Are there duplicates?
  - Is the array sorted?
  - What are the constraints? (size of N)
  - Integer overflow possible?

### Step 2: Think Out Loud — Brute Force First (2 min)
- Always state the brute force solution first
- "Naive approach is O(N²) — let me optimize"
- Shows you understand the problem before jumping to tricks

### Step 3: Optimize (5-10 min)
- Identify the bottleneck (usually a nested loop)
- Think about which data structure eliminates it
- Use patterns: sliding window, two pointers, HashMap, monotonic stack, etc.

### Step 4: Code (15-20 min)
- Write clean code, name variables clearly
- Handle edge cases before coding main logic
- Talk while coding — explain what each block does

### Step 5: Test (3-5 min)
- Trace through your code with the given example
- Then test an edge case: empty input, single element, all duplicates
- Calculate final time and space complexity

---

## Pattern Recognition — Most Important Skill

Recognizing the pattern is 80% of solving the problem.

| Pattern | When to Use | Example Problems |
|---|---|---|
| **Sliding Window** | Contiguous subarray/substring with a condition | Longest substring without repeating chars, Max sum subarray of size K |
| **Two Pointers** | Sorted array, pair sum, remove duplicates | Two Sum (sorted), 3Sum, Container with most water |
| **Fast & Slow Pointers** | Cycle detection in linked list / array | Linked list cycle, Find duplicate number, Middle of linked list |
| **Binary Search** | Sorted array OR monotonic condition | Search in rotated array, Kth smallest, Minimum in rotated array |
| **BFS** | Shortest path, level-order, unweighted graph | Word ladder, Rotting oranges, Level order traversal |
| **DFS / Backtracking** | All paths, permutations, combinations, subsets | Subsets, Permutations, N-Queens, Word Search |
| **Dynamic Programming** | Overlapping subproblems + optimal substructure | Longest common subsequence, 0/1 Knapsack, Coin change |
| **Greedy** | Local optimal → global optimal | Jump game, Meeting rooms, Interval scheduling |
| **Monotonic Stack** | Next greater/smaller element, span problems | Daily temperatures, Largest rectangle in histogram |
| **HashMap / HashSet** | O(1) lookup, counting frequency, grouping | Two Sum, Group anagrams, Longest consecutive sequence |
| **Heap / Priority Queue** | Top K elements, Kth largest, merge K sorted | Kth largest element, Top K frequent, Merge K sorted lists |
| **Trie** | Prefix search, word dictionary, autocomplete | Implement trie, Word search II, Replace words |
| **Union Find** | Connected components, cycle detection in graph | Number of islands, Redundant connection, Accounts merge |
| **Divide & Conquer** | Split problem in half recursively | Merge sort, Quick sort, Count inversions |

---

## Topic-Wise Must-Know Problems

### Arrays & Strings
| Problem | Difficulty | Pattern | Company |
|---|---|---|---|
| Two Sum | Easy | HashMap | Everywhere |
| Best Time to Buy and Sell Stock | Easy | Sliding window / one pass | Everywhere |
| Contains Duplicate | Easy | HashSet | Everywhere |
| Product of Array Except Self | Medium | Prefix/suffix product | Google, Amazon |
| Maximum Subarray (Kadane's) | Medium | DP / Greedy | Everywhere |
| Maximum Product Subarray | Medium | DP | Google, Amazon |
| Find Minimum in Rotated Sorted Array | Medium | Binary search | Everywhere |
| Search in Rotated Sorted Array | Medium | Binary search | Everywhere |
| 3Sum | Medium | Two pointers | Everywhere |
| Container with Most Water | Medium | Two pointers | Amazon, Google |
| Trapping Rain Water | Hard | Two pointers / Monotonic stack | Google, Amazon, Meta |
| Sliding Window Maximum | Hard | Monotonic deque | Google |
| Longest Substring Without Repeating Chars | Medium | Sliding window + HashMap | Everywhere |
| Minimum Window Substring | Hard | Sliding window | Google, Meta |
| Group Anagrams | Medium | HashMap + sorting | Everywhere |
| Longest Consecutive Sequence | Medium | HashSet | Meta, LinkedIn |
| Rotate Image | Medium | In-place matrix | Amazon, Google |
| Spiral Matrix | Medium | Simulation | Amazon, Microsoft |
| Set Matrix Zeroes | Medium | In-place | Amazon |
| Valid Palindrome | Easy | Two pointers | Everywhere |

### Linked Lists
| Problem | Difficulty | Pattern | Company |
|---|---|---|---|
| Reverse Linked List | Easy | Iterative / Recursive | Everywhere |
| Merge Two Sorted Lists | Easy | Two pointers | Everywhere |
| Reorder List | Medium | Fast/slow + reverse | Google, Amazon |
| Remove Nth Node from End | Medium | Two pointers (gap = N) | Everywhere |
| Linked List Cycle | Easy | Fast/slow pointers | Everywhere |
| Linked List Cycle II (start of cycle) | Medium | Fast/slow pointers + math | Amazon, Google |
| Merge K Sorted Lists | Hard | Heap / Divide & conquer | Google, Amazon, Meta |
| Reverse Nodes in K-Group | Hard | Recursion | Google, Meta |
| LRU Cache | Medium | HashMap + Doubly linked list | Everywhere |
| Copy List with Random Pointer | Medium | HashMap | Amazon, Meta |

### Trees & Binary Search Trees
| Problem | Difficulty | Pattern | Company |
|---|---|---|---|
| Invert Binary Tree | Easy | DFS recursion | Everywhere |
| Maximum Depth of Binary Tree | Easy | DFS | Everywhere |
| Same Tree | Easy | DFS | Everywhere |
| Subtree of Another Tree | Easy | DFS | Amazon |
| Lowest Common Ancestor (BST) | Easy | BST property | Everywhere |
| Lowest Common Ancestor (Binary Tree) | Medium | DFS | Meta, Google |
| Level Order Traversal | Medium | BFS | Everywhere |
| Binary Tree Right Side View | Medium | BFS | Meta, Amazon |
| Count Good Nodes in Binary Tree | Medium | DFS | Google |
| Validate Binary Search Tree | Medium | DFS with range | Everywhere |
| Kth Smallest Element in BST | Medium | In-order traversal | Everywhere |
| Construct Binary Tree from Preorder+Inorder | Medium | Divide & conquer | Everywhere |
| Binary Tree Maximum Path Sum | Hard | DFS post-order | Google, Meta |
| Serialize / Deserialize Binary Tree | Hard | BFS / DFS | Google, Amazon, Meta |
| Word Search II | Hard | Trie + DFS | Google, Meta |

### Graphs
| Problem | Difficulty | Pattern | Company |
|---|---|---|---|
| Number of Islands | Medium | BFS / DFS | Everywhere |
| Clone Graph | Medium | BFS + HashMap | Meta, Google |
| Pacific Atlantic Water Flow | Medium | DFS from borders | Google |
| Course Schedule (Cycle Detection) | Medium | Topological sort / DFS | Everywhere |
| Course Schedule II (Order) | Medium | Topological sort (Kahn's) | Google, Amazon |
| Number of Connected Components | Medium | Union Find / DFS | LinkedIn, Google |
| Redundant Connection | Medium | Union Find | Meta |
| Word Ladder | Hard | BFS (shortest path) | Amazon, Google |
| Alien Dictionary | Hard | Topological sort | Google, Meta, Airbnb |
| Accounts Merge | Medium | Union Find | Google |
| Rotting Oranges | Medium | BFS (multi-source) | Amazon, Google |
| Walls and Gates | Medium | BFS (multi-source) | Meta |
| Cheapest Flights Within K Stops | Medium | Bellman-Ford / Dijkstra | Google |
| Network Delay Time | Medium | Dijkstra | Amazon, Google |

### Dynamic Programming
| Problem | Difficulty | Pattern | Company |
|---|---|---|---|
| Climbing Stairs | Easy | 1D DP | Everywhere |
| House Robber | Medium | 1D DP | Everywhere |
| House Robber II (circular) | Medium | 1D DP twice | Everywhere |
| Longest Palindromic Substring | Medium | 2D DP / expand around center | Amazon, Google |
| Palindromic Substrings (count) | Medium | Expand around center | Meta |
| Coin Change (min coins) | Medium | 1D DP | Everywhere |
| Coin Change II (count ways) | Medium | 1D DP | Google |
| Longest Increasing Subsequence | Medium | DP + Binary search | Google, Amazon |
| Unique Paths | Medium | 2D DP | Everywhere |
| Jump Game | Medium | Greedy | Everywhere |
| Jump Game II (min jumps) | Medium | Greedy | Google |
| Word Break | Medium | 1D DP + HashMap | Everywhere |
| Combination Sum IV | Medium | 1D DP | Google |
| 0/1 Knapsack | Medium | 2D DP | Amazon, Microsoft |
| Decode Ways | Medium | 1D DP | Meta, Amazon |
| Longest Common Subsequence | Medium | 2D DP | Google, Amazon |
| Edit Distance | Hard | 2D DP | Google, Meta |
| Burst Balloons | Hard | Interval DP | Google |
| Regular Expression Matching | Hard | 2D DP | Google, Meta |

### Heap / Priority Queue
| Problem | Difficulty | Pattern | Company |
|---|---|---|---|
| Kth Largest Element in Array | Medium | Min-heap of size K | Everywhere |
| Top K Frequent Elements | Medium | Min-heap / bucket sort | Everywhere |
| Find Median from Data Stream | Hard | Two heaps (max + min) | Google, Amazon |
| Merge K Sorted Lists | Hard | Min-heap | Google, Amazon, Meta |
| Task Scheduler | Medium | Greedy + Max-heap | Meta |
| Design Twitter (Top 10 tweets) | Medium | Heap + linked list | Google |
| K Closest Points to Origin | Medium | Max-heap of size K | Amazon, Meta |

### Intervals
| Problem | Difficulty | Pattern | Company |
|---|---|---|---|
| Meeting Rooms (can attend all?) | Easy | Sort + check overlap | Everywhere |
| Meeting Rooms II (min rooms) | Medium | Min-heap / two pointers | Google, Amazon, Meta |
| Merge Intervals | Medium | Sort + merge | Everywhere |
| Insert Interval | Medium | Linear scan | Everywhere |
| Non-Overlapping Intervals | Medium | Greedy (sort by end) | Google |
| Employee Free Time | Hard | Heap / merge intervals | Google |

### Backtracking
| Problem | Difficulty | Pattern | Company |
|---|---|---|---|
| Subsets | Medium | Backtracking | Everywhere |
| Subsets II (with duplicates) | Medium | Backtracking + sort | Everywhere |
| Permutations | Medium | Backtracking | Everywhere |
| Permutations II (with duplicates) | Medium | Backtracking + sort | Everywhere |
| Combination Sum | Medium | Backtracking | Everywhere |
| Combination Sum II | Medium | Backtracking + sort | Everywhere |
| Word Search | Medium | DFS + backtrack on grid | Everywhere |
| N-Queens | Hard | Backtracking | Google, Meta |
| Palindrome Partitioning | Medium | Backtracking + DP | Everywhere |
| Letter Combinations of Phone Number | Medium | Backtracking | Everywhere |

### Binary Search (Advanced)
| Problem | Difficulty | Pattern | Company |
|---|---|---|---|
| Binary Search | Easy | Classic | Everywhere |
| Search a 2D Matrix | Medium | Binary search on flattened | Everywhere |
| Koko Eating Bananas | Medium | Binary search on answer | Google, Amazon |
| Minimum in Rotated Sorted Array | Medium | Binary search | Everywhere |
| Find Peak Element | Medium | Binary search | Google |
| Time-Based Key-Value Store | Medium | Binary search on sorted list | Everywhere |
| Median of Two Sorted Arrays | Hard | Binary search | Everywhere |
| Split Array Largest Sum | Hard | Binary search on answer | Google |

### Stack & Queue
| Problem | Difficulty | Pattern | Company |
|---|---|---|---|
| Valid Parentheses | Easy | Stack | Everywhere |
| Min Stack | Medium | Stack with min tracking | Everywhere |
| Daily Temperatures | Medium | Monotonic stack | Everywhere |
| Car Fleet | Medium | Monotonic stack | Google |
| Largest Rectangle in Histogram | Hard | Monotonic stack | Google, Amazon |
| Evaluate Reverse Polish Notation | Medium | Stack | Amazon |
| Generate Parentheses | Medium | Backtracking / Stack | Google |
| Decode String | Medium | Stack | Google, Amazon |

### Math & Bit Manipulation
| Problem | Difficulty | Pattern | Company |
|---|---|---|---|
| Single Number | Easy | XOR | Everywhere |
| Number of 1 Bits | Easy | Bit manipulation | Everywhere |
| Counting Bits | Easy | DP + bits | Everywhere |
| Missing Number | Easy | XOR / Gauss formula | Everywhere |
| Reverse Bits | Easy | Bit manipulation | Everywhere |
| Power of Two | Easy | Bit manipulation | Everywhere |
| Sum of Two Integers (no +) | Medium | Bit manipulation | Meta |
| Multiply Strings | Medium | Grade school multiplication | Google |

---

## Complexity Cheat Sheet

### Time Complexity

| Complexity | Name | Example |
|---|---|---|
| O(1) | Constant | HashMap get/put, array index access |
| O(log N) | Logarithmic | Binary search, heap insert |
| O(N) | Linear | Single loop, BFS/DFS |
| O(N log N) | Log-linear | Merge sort, heap sort, sorting |
| O(N²) | Quadratic | Nested loops, bubble sort |
| O(2^N) | Exponential | Backtracking subsets |
| O(N!) | Factorial | Backtracking permutations |

### Data Structure Operations

| Data Structure | Access | Search | Insert | Delete |
|---|---|---|---|---|
| Array | O(1) | O(N) | O(N) | O(N) |
| Linked List | O(N) | O(N) | O(1) | O(1) |
| Stack / Queue | O(N) | O(N) | O(1) | O(1) |
| HashMap / HashSet | O(1) avg | O(1) avg | O(1) avg | O(1) avg |
| Binary Search Tree | O(log N) avg | O(log N) avg | O(log N) avg | O(log N) avg |
| Heap (Min/Max) | O(1) peek | O(N) | O(log N) | O(log N) |
| Trie | — | O(M) | O(M) | O(M) |

M = length of string/key

### Sorting Algorithms

| Algorithm | Best | Average | Worst | Space | Stable? |
|---|---|---|---|---|---|
| Merge Sort | O(N log N) | O(N log N) | O(N log N) | O(N) | Yes |
| Quick Sort | O(N log N) | O(N log N) | O(N²) | O(log N) | No |
| Heap Sort | O(N log N) | O(N log N) | O(N log N) | O(1) | No |
| Counting Sort | O(N+K) | O(N+K) | O(N+K) | O(K) | Yes |
| Tim Sort (Java default) | O(N) | O(N log N) | O(N log N) | O(N) | Yes |

---

## Company-Specific DSA Focus

| Company | Favourite Topics | Notes |
|---|---|---|
| **Google** | Graphs, DP, Trees, String manipulation | Hard problems, multiple optimal solutions expected |
| **Meta/Facebook** | Trees, Graphs, Arrays, System + DSA combo | Clean code, handle edge cases explicitly |
| **Amazon** | Arrays, Strings, Trees, OOP design | 2 LC rounds + 1 design round, LP questions mixed in |
| **Microsoft** | Trees, Graphs, Arrays, LinkedList | Medium difficulty, explain approach clearly |
| **Uber** | Graphs (geo), Arrays, Heap | Often scenario-based (driver matching, trip problems) |
| **LinkedIn** | Graphs (social network), Strings, DP | Graph traversal very common |
| **Airbnb** | Arrays, Strings, Backtracking | Clean code culture, readability matters |
| **Stripe/PayPal** | HashMap, Arrays, OOP design | Often real-world scenario (billing, transactions) |
| **Apple** | Trees, Arrays, Bit manipulation | Focus on correctness and edge cases |

---

## Quick Checklist — Before Any DSA Interview

- [ ] Two Sum and its variants (HashMap pattern)
- [ ] Sliding window — fixed size and dynamic
- [ ] Binary search on answer (Koko, Split array)
- [ ] BFS for shortest path, DFS for all paths
- [ ] Topological sort — both DFS and Kahn's (BFS)
- [ ] Union Find — path compression + union by rank
- [ ] Dijkstra's algorithm — shortest path weighted graph
- [ ] Monotonic stack — next greater element, histogram
- [ ] Two heaps — median from data stream
- [ ] 0/1 Knapsack and Unbounded Knapsack
- [ ] LCS and Edit Distance (2D DP)
- [ ] Trie — insert, search, startsWith
- [ ] LRU Cache — HashMap + Doubly Linked List
- [ ] Backtracking template — choose / explore / unchoose
- [ ] Fast & Slow pointers — cycle detection, find middle
- [ ] XOR tricks — single number, missing number
- [ ] Bit manipulation — n & (n-1) clears lowest set bit
