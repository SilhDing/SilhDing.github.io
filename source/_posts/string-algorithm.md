---
title: Algorithms of String
date: 2020-04-18 21:15:09
tag:
    - "algorithm"
categories:
    - "technique"
---

In this post, we want to study and summarize some well-known algorithms for Strings.

## String Sorts

### LSD

If we want to sort some items and each item is associated with a key (or ID). We could use *key-indexed* counting to sort, which is very efficient. This method also indicates that we could follow the same idea to sort **fixed-length** strings, which is called ***least-significant-digit first (LSD)*** sort.

The basic idea is to conduct key-indexed counting for w times, where w is the length of each string.

```Java
public class LSD {
    public static void sort(String[] a, int W) {
        // W is the length of each string in a
        int N = a.length;
        int R = 256;
        String[] aux = new String[N];

        for (int d = W - 1; d >= 0; d--) {
            // Sort by key-indexed counting on dth char.

            int[] count = new int[R+1];
            // Computer frequency counts.
            for (int i = 0; i < N; i++) {
                count[a[i].charAt(d) + 1] ++;
            }

            // Transform counts to indices.
            for (int r = 0; r < R; r++) {
                count[r+1] += count[r];
            }

            // Distribute.
            for (int i = 0; i < N; i++) {
                aux[count[a[i].charAt(d)]++] = a[i];
            }

            // Copy back.
            for (int i = 0; i < N; i++) {
                a[i] = aux[i];
            }
        }
    }

    public static void main(String[] args) {
        String[] a  = {"43FRFN", "423FFE", "D439JN", "10NE2E",
                        "DDERFR", "340FEW", "104FDS", "R4053N"};
        LSD.sort(a, 6);
        System.out.println(Arrays.toString(a));
    }
}
```

Time complexity: `O(W(N + R))`, where `N` is the number of all string to be sorted, `R` is the radix (number of characters in alphabet) and `W` is the width of each string.

Space complexity: `O(N + R)`, as we use two additional arrays.

Remember that there are some requirements when applying this method:

1. Each string has the same length (certainly we can modify this method to accommodate this case, but we have other algorithms that work well for this scenario);
2. We know the radix of the alphabet of these strings (in this case, 256).

### MSD

LSD algorithm listed above only works for fix-length strings. Here we want to implement an algorithm that works for a general purpose: *most-significant-digit-first (MSD)*.

We still follow the basic idea of key-indexed counting to complete this algorithm. Now, as the algorithm's name indicates, we will start sorting based on the most significant digits. So, if we have sorted them based on the first digit, what should we do next? **We can group (or split) these strings and sort in each group based on other digits,** which requires a recursive algorithm.

Remember, this algorithm does not require all strings with same length. What should we do if, in a group, there are string which are shorter than others and we have reached the end of these short strings during key-indexed counting? Okay, we know that shorter strings here should be sorted first, so here is the solution:

Implement a new `toChar()` method that can convert from an indexed string character to an array index that returns -1 if the specified character position is pass the end of the string. Then, we just add 1 to each returned value, so get a nonnegative `int` that we can use to index `count[]`. It means we have `R+1` possible character values at each string position:

- `0` to signify the end of the string;
- `1` for the first alphabet character;
- `2` for the second alphabet character;

With all these practices above, now it is easy to write down the real code.

```Java
public class MSD {

    private static int R = 256;  // radix
    private static final int M = 0;  // cutoff for small subarrays
    private static String[] aux; // auxiliary array for distribution

    public static void sort(String[] a) {
        int N = a.length;
        aux = new String[N];
        sort(a, 0, N-1, 0);
    }

    private static void sort(String[] a, int lo, int hi, int d) {
        // Cut-off if the number of string is too small
        if (lo + M >= hi) {
            Insertion.sort(a, lo, hi, d);
            return ;
        }

        int[] count = new int[R+2];
        // Compute frequency counts.
        for (int i = lo; i <= hi; i++) {
            count[charAt(a[i], d) + 2] ++;
        }

        // Transform counts to indices.
        for (int i = 0; i < R +1; i++) {
            count[i+1] += count[i];
        }

        // Distribute.
        for (int i = lo; i <= hi; i++) {
            aux[count[charAt(a[i], d) + 1]++] = a[i];
        }

        // Copy back.
        for (int i = lo; i <= hi; i++) {
            // Consider why i - lo here: because we only sort hi- lo + 1 strings, starting with index 0 in aux
            a[i] = aux[i - lo];
        }

        // Recursive call: process to the next d for each split
        for (int r = 0; r < R; r++) {
            sort(a, lo + count[r], lo + count[r+1] - 1, d + 1);
        }

    }

    private static int charAt(String s, int index) {
        if (index < s.length())
            return s.charAt(index);
        else
            return -1;
    }

    public static void main(String[] args) {
        String[] a = {"she", "sells", "seashells", "by", "the", "sea",
                        "shore", "the", "shells", "she", "sells", "are", "surely", "seashells"};
        sort(a);
        System.out.println(Arrays.toString(a));
        String[] b = {"aaaaaa","aaaaaa","aaaaaa","aaaaaa","aaaaaa","aaaaaa","aaaaaa","aaaaaa"};
        sort(b);
    }
}
```

Note that we add a "cutoff-for-small-subarrays" before the sort really begins. why do we need this?

As we can see, the method quickly divides the array to be sorted into small subarrays. But this is also a double-edged sword: we are certain to have to handle huge numbers of tiny subarrays, so we had better be sure that we handle them efficiently. Each sort involves initializing the 258 (if `R = 256`) entries of the `count[]` array to `0` and transforming them all to indices. Accordingly, the switch to insertion sort for small subarrays is a must for MSD string sort (in the code I simply use a `Insertion.sort()` for this part).

Please also note another pitfall of MSD sort algorithm: it can be relatively slow for subarrays containing large numbers of equal keys. ***The worst case for MSD string sorting is when all key are equal***: the algorithm cannot make smaller subarrays after each iteration (d). In this case, out program will have to examine every character in all input strings, and the running time is linear in the number of characters in the data (like LSD string sort).

In many cases, we should treat the input strings are randomly generated. In this case, MSD string sort examines just enough characters to distinguish among the keys, and the running time is sub-linear in the number of characters in the data.

### Three-way string quicksort

This algorithm borrows the idea from quick sort.

```Java
public class Quick3string {

    public static void sort(String[] a) {
        sort(a, 0, a.length - 1, 0);
    }

    private static void sort(String[] a, int lo, int hi, int d) {
        if (lo >= hi) return;

        int lt = lo, gt = hi;
        int i = lo + 1;
        int v = charAt(a[lo], d);

        while (i <= gt) {
            int t = charAt(a[i], d);

            if (t < v) exch(a, lt++, i++);
            else if (t > v) exch(a, i, gt--);
            else i ++;
        }

        sort(a, lo, lt - 1, d);
        if (v >= 0) sort(a, lt, gt, d + 1);
        sort(a, gt + 1, hi, d);
    }

    private static int charAt(String s, int index) {
        if (index < s.length())
            return s.charAt(index);
        else
            return -1;
    }

    private static void exch(String[] a, int i, int j) {
        String tmp = a[i];
        a[i] = a[j];
        a[j] = tmp;
    }

    public static void main(String[] args) {
        String[] a = {"she", "sells", "seashells", "by", "the", "sea",
                "shore", "the", "shells", "she", "sells", "are", "surely", "seashells"};
        sort(a);
        System.out.println(Arrays.toString(a));
    }
}
```

This algorithm will always partition the array into three subarrays, which is different from MSD as MSD might generate many subarrays and many of them might be small or even empty subarrays. It can adapt well to handling equal keys, keys with long common prefixes. **It will not use extra space, just like quick sort**.

An interesting fact will be found when you try to compare this algorithm with standard quicksort. Indeed, one way to think 2-way string quick sort is ***as a way for standard quicksort to keep track of leading characters that are known to be equal***. Note: no algorithm can beat 3-way string quicksort by more than a constant factor.

## Tries

```Java
public class TrieSTNew<Value> {

    private static int R = 256;
    private Node root;

    private static class Node {
        private Object val;
        private Node[] next = new Node[R];
    }

    // When finding the node, we use recursion rather than iteration.
    public Value get(String key) {
        Node node = get(root, key, 0);
        if (node == null) return null;
        return (Value) node.val;
    }

    private Node get(Node x, String key, int d) {
        // Return value associated with key in the subtrie rooted at x.
        if (x == null) return null;
        // Remember, right now we are at the node corresponding to char at index d-1
        if (d == key.length()) return x;
        char c = key.charAt(d);
        return get(x.next[c], key, d + 1);
    }

    public void put(String key, Value value) {
        root = put(root, key, value, 0);
    }

    public Node put(Node x, String key, Value value, int d) {
        if (x == null) x = new Node();
        // Remember, right now we are at the node corresponding to char at index d-1
        if (d == key.length()) {
            x.val = value;
            return x;
        }
        char c = key.charAt(d);
        x.next[c] = put(x.next[c], key, value, d + 1);
        return x;
    }

	public static void main(String[] args) {
		TrieSTNew<Integer> trie = new TrieSTNew<>();
		trie.put("dede", 0);
		trie.put("de", 0);
		trie.put("deeffe", 0);

		for (String str: trie.keys()) {
			System.out.print( str+" ");
		}
		System.out.println();
		System.out.println(trie.root);
	}
}
```
### Collecting keys

Now, we are going to implement some methods for collecting keys. The above code contains some basic operations including `get` and `put`. Now we will continue to implement some method.

```Java
public Iterable<String> keys() {
    return keysWithPrefix("");
}

public Iterable<String> keysWithPrefix(String pre) {
    Queue<String> queue = new LinkedList<>();
    collect(this.get(root, pre, 0), pre, queue);
    return queue;
}

public void collect(Node x, String pre, Queue<String> q) {
    // Starting with Node x, find all keys
    if (x == null) return ;
    if (x.val != null) q.add(pre);
    for (char c = 0; c < R; c++)
        collect(x.next[c], pre + c, q);
}
```

### Wildcard match

Wildcard match: to implement `keysThatMatch()` method.

```Java
public Iterable<String> keysThatMatch(String pat) {
    Queue<String> queue = new LinkedList<>();
    collect(root, "", pat, queue);
    return queue;
}

private void collect(Node x, String pre, String pat, Queue<String> queue) {
    if (x == null) return ;
    int len = pre.length();

    if (len == pat.length()) {
        if (x.val != null) queue.add(pre);
        return ;
    }

    char next = pat.charAt(len);
    for (char c = 0; c < R; c ++) {
        if (c == next || c == '.')
            collect(x.next[c], pre + c, pat, queue);
    }
}
```

### Longest prefix

Now we also want to method `longestPrefixOf(String s)` to find the longest string which is a prefix of a given string. The principle is quite similar: go down the trie with the given string; once we find a node with a non-null value, we just update the length.

```Java
public String longestPrefixOf(String s) {
    int length = search(root, s, 0, 0);
    return s.substring(0, length);
}

private int search(Node x, String s, int d, int length) {
    if (x == null) return length;
    if (x.val != null) length = d;
    if (d == s.length()) return length;
    char c = s.charAt(d);
    return search(x.next[c], s, d + 1, length);
}
```

### Deletion

What if we want to delete a key? Apparently, the first step is to find the node corresponding to the input string and then set the value to `null`. However, we need to remove many nodes all the way down. This can be easily implemented with the recursive method.

```Java
public void delete(String key) {
    root = delete(root, key, 0);
}

public Node delete(Node x, String key, int d) {
    if (x == null) return null;

    if (d == key.length()) {
        // Find the node representing the key
        x.val = null;
    } else {
        // Keep going down and find the node.
        char c = key.charAt(d);
        x.next[c] = delete(x.next[c], key, d + 1);
    }

    if (x.val != null) return x;

    for (char c = 0; c < R; c ++) {
        // Check if x have a non-null link
        if (x.next[c] != null) return x;
    }
    return null;
}
```

### Performance

It is easy to prove the time complexity of this algorithm. What about space?

The number of links in a trie us between `RN` and `RNw`, where w is the average key length.

## TSTs

Ternary search tries (TSTs) are used to avoid the excessive space cost associated with R-way tries. In a TST, each node has a character, three links, and a value.The three links corresponds to keys whose current characters are less than, equal, or greater than the node's character.

```Java
public class TSTNew<Value> {
    private Node root;

    private class Node {
        char c;
        Node left, mid, right;
        Value value;
    }

    public Value get(String key) {
        Node node = get(root, key, 0);
        if (node == null) return null;
        // Note: value could be null
        return node.value;
    }

    private Node get(Node x, String key, int d) {
        if (x == null) return null;

        char c = key.charAt(d);
        if (c < x.c) {
            // Go to the left side.
            return get(x.left, key, d);
        } else if (c > x.c) {
            // Go to the right side.
            return get(x.right, key, d);
        } else if (d < key.length() - 1) {
            // Have not reached the end of key, need to go down.
            return get(x.mid, key, d + 1);
        } else {
            // Have reached the end of the key.
            return x;
        }
    }

    public void put(String key, Value val) {
        root = put(root, key, val, 0);
    }

    private Node put(Node x, String key, Value val, int d) {
        char c = key.charAt(d);
        if (x == null) {
            x = new Node();
            x.c = c;
        }

        if (c < x.c) {
            // Go to the left side.
            x.left = put(x.left, key, val, d);
        } else if (c > x.c) {
            // Go to the rigth side.
            x.right = put(x.right, key, val, d);
        } else if (d < key.length() - 1) {
            // Have not reached the end of the key.
            x.mid = put(x.mid, key, val, d + 1);
        } else {
            // Have reached the end of the key.
            x.value = val;
        }

        return x;
    }
}
```

The number of links in a TST build from N string keys of average length `w` is between `3N` and `3Nw`.

## Substring Search

### Brute-force

```Java
public class StringSearch {
    // This is the brute-force substring search

    public static int search(String pat, String txt) {
        int M = pat.length();
        int N = txt.length();

        for (int i = 0; i <= N - M; i++) {
            int j;
            for (j = 0; j < M; j ++) {
                if (txt.charAt(i + j) != pat.charAt(j))
                    break;
            }
            if (j == M) return i;
        }
        return N;
    }

    public static void main(String[] args) {
        System.out.println(search("ABRA", "ABACADABRAC"));
        System.out.println(search("ABRA", "ABACADABRBC"));

    }
}
```

Time complexity: `O(MN)`. Below is an alternative solution.

```Java
public static int search2(String pat, String txt) {
    int M = pat.length();
    int N = txt.length();

    int i, j;
    for (i = 0, j = 0; i < N && j < M; i ++) {
        if (txt.charAt(i) == pat.charAt(j)) {
            j++;
        } else {
            i -= j;
            j = 0;
        }
    }

    if (j == M) return i - M;
    else return N;

}
```

### KMP

Here I will introduce two versions of KMP algorithms.

#### 2D Version with DFA

As we mentioned in the brute-force substring match algorithm (the alternate one), if we find a mismatch at some point, `j` has to set to be `0` and `i` will also decremented. This is the main reason that the running time of that algorithm is high.

Is there a way which avoids decrementing the value of 'j'. This is what KMP algorithm tries to achieve. The basic idea behind the algorithm discovered by Knuth, Morris, and Pratt is this: whenever we detect a mismatch, *we already know some of the characters in the text*, since they matched the pattern characters prior to the mismatch.

In KMP substring search, we never back up the text pointer `i`, and we use an array `dfa[][]` to record how far to back up the pattern pointer `j` when a mismatch is detected. For every character `c`, `dfa[c][j]` is the pattern position to compare with next text position after comparing `c` with `pat.charAt(j)`. During the search, **`dfa[txt.charAt(i)][j]` is the pattern position to compare with `txt.charAt(i)` with `pat.charAt(j)`**. Thus, `dfa[pat.charAt(j)][j]` is always `j+1`.

Once we have computed DFA, we then can easily search the match with DFA. You may ask, why we have to call it "DFA"?

In fact, "DFA" represents *deterministic finite-state automation*. For example, for a pattern `A B A B A C`, the DFA will be (we will later study how can we construct DFA):

	   A B C D E F
	j  0 1 2 3 4 5
	   -----------
	A |1 1 3 1 5 1
	B |0 2 0 4 0 4
	C |0 0 0 0 0 6

Please note that here we assume there are only three characters (A, B, and C) in the alphabet. Below is the graphical representation of this DFA.

![dfa_graph](dfa_graph.jpg)

We then could easily use this to finish the matching process. The input text string is "B C B A A B A C A A B A B A C A A":

![dfa_process](dfa_process.jpg)

Below is the code corresponding to the process showed in the picture.

```Java
public class KMP {

    private String pat;
    private int[][] dfa;

    public KMP(String pat) {
        this.pat = pat;
        int M = pat.length();
        int R = 256;
        dfa = new int[R][M];
        // TODO: construct the DFA array    
    }

    public int search(String txt) {
        int i, j;
        int N = txt.length(), M = this.pat.length();
        for(i = 0, j = 0; i < N && j < M; i++) {
            j = dfa[txt.charAt(i)][j];
        }
        if (j == M) return i - M;
        else return N;
    }

}
```

Up to now, you must feel the logic is natural and comfortable. Now, we have to know how can we construct DFA array with a given pattern string, which is much trickier and interesting.

When we have a mismatch at `pat.charAt(j)`, our interest is in knowing in what state the DFA *would be* if we were to back up the next index and rescan the text characters that we just saw after shifting to the right one position (remember, `i` is always incremented by `1` after a match or mismatch).

What should the DFA do with the next character? ***Exactly what it would have done if we had backed up***, except if it finds a match with `pat.charAt(j)`, when it should go to state `j+1`. Certainly, this makes sense and is straightforward.

So, the question here is, what is the back up state X? before this question, we must know what exactly happens (or, say, what we should do) when backing up to state X. Here are two question to discussed about:

> Q: What we actually do after backing up to state `X`?

If we back up to state X after a mismatch, keep `i` not changed, and set `j` to X. The definition of X should guarantee that **all chars from index 0 to `j - 1` (i.e., `X - 1`) are matched with chars prior to `i` in the text string**. We now, we just need to compare the char of index `j` (`X`) in the pattern string and the char of index `i` in the text string, this may proceed to a nigger state or a level state. Because our DFA is partially completed already, and `X < j`, so we can use the DFA now! which is: `dfa[c][X]`, where `c` is the current text char at index `i`. And then, according to the definition of DFA, we get a new position for `j` (the result of `dfa[c][X]`), and `i` is incremented by 1.

***Thus, we can copy the entry values in state `X` to state `j`, except when we see a match: `dfa[pat.charAt(j)][j] = j + 1`.***

I know this is kind of confusing, so let's look into 2 example. In both these examples, we use pattern string `A B A B A C`.

**~ Example 1**

Say we have completed the first column of DFA (the first column of this DFA is in fact the complete DFA for pattern string `A`), and we are building the second column. As mentioned earlier, we should back up (exception there is a match, and we will handle this case specially) to state `X`. `X` for the second column (`j = 1`) is `0` (we will explain why `X` is this value later).

	             (i)
	txt string: A A C ...
	pat string: A B A ...
	             (j)

We can see that when state is `1` (`j = 1`) we have a mismatch. As we said previously, we need to remain the value of `i`, and set `j` to `0`.

	             (i)
	txt string: A A C...
	pat string:   A B A...
	             (j)
	             (X)

We have backed up the `X`, and we have not finished our job. We could only make sure that all chars from index `0` to `X - 1` is pattern string are matched (of course, it is none in this example), but how about char on index `X` (`i` for text string)? We cannot make sure it is also match.

In fact, because we have finished the first column of DFA, and `j` is `0` now, and you may also notice that ***we are actually doing a match***! All of these indicate that we can use the partially completed DFA to determine a new state, which is given by `dfa['A'][X]`. The first `A` indicates the char on index `i` in text string. If the text string is `A B ...`, then use `dfa['B'][X]`. As we are building the second column of DFA, we might need a loop: `dfa[][j] = dfa[][X]` (note `j` here is the original `y`).

Continue this example. `dfa['A'][X]`, which is also `dfa['A'][0]` will return `1`. Thus, we set `j` to `1` and increment `i` by 1.

	               (i)
	txt string: A A C...
	pat string:   A B A...
	               (j)

**~ Example 2**

You may notice that the example 1 does not show that **all chars from index 0 to `X - 1` are matched with chars prior to `i` in the text string**. No worry, we can give you another example.

Suppose now we have a mismatch at index `j = 3`, and `X` is `1`. For example, it happens when the text string is `A B A C ...`

	                 (i)
	txt string: A B A C A...
	pat string: A B A B A...
	                 (j)

As we discussed previously, we need to remain the value of `i`, and set `j` to `X`, which is `1`.

	                 (i)
	txt string: A B A C A ...
	pat string:     A B A B A ...
	                 (j)
	                 (X)

We can see all chars from index `0` to `X - 1`, in fact, `A`, match with txt string. So, we only need to care about the chars after `X`, and we can just use value of `dfa[C][X]` as the DFA is partially built. The value given by it is `0`.

	                   (i)
	txt string: A B A C A ...
	pat string:         A B A B A ...
	                   (j)

I believe these two examples can help you understand the process that we build the DFA array. The key to understand the process is to be aware that state `X` *can make sure all chars prior to index `X` in pattern string have will be matched if we set `j` to `X` and do not change `i`*.

There is another way to understand `X`, and it will help you understand the next question.

For a given string, we define its ***prefix set*** as the set of all its prefix (exclude the string itself) and ***suffix set*** as the set of all its suffix (exclude the string itself). **`X` will be the length of the longest string in the intersection set of the string's prefix and suffix set**. For example, for string `ABA`, longest string is `A`, and `X` is `1`; for `ABAB`, `X` will be `2`.

We have image the you are match two same strings, while sliding one to the left and another one to right. Once the overlapping matches, we get the `X` value. Take `ABAB` as an example:

	← A B A B
	      A B A B →

It should be easy to derive this representation once you have understand the two examples above.

In fact, this is also the way to compute the value of `X`. However, we don't want to use this naive way, and here comes our second question.

> Q: How can we compute the value of `X`?

At first, let's talk about what is the initial value of `X` when we just start to build the DFA array.

- If we mismatch at state `0`, of course we should go back to state `0` (`j = 0`). This is totally understandable;
- If we mismatch at state `1`, we could only back up to `0` as well as we cannot stay the same state.

What if we mismatch at other positions? The method is, every time when we have finished one column of DFA, **we need to update the value of `X`**.

Say we now have finished the first three columns of the DFA array and are building the 4th column. The prefix with length of 3 is `ABA`, and the `X` value during the construction of 4th column is `1`:

	← A B A
	      A B A →
	       (X)

Apparently, value of `X` is the length that we can self-matched, as showed above. Say now we also have finished the 4th column, and the prefix length should be `4`. Then, it is necessary to update the value of `X`:

	   (new char added)
	        ↓
	← A B A B
	      A B A B →
	       (X)  ↑
	          (new char added)

A critical observation is, to determine the new value of `X` (or, say, the new length of self-matching), ***we only need to match the new char added with the char on index of previous `X`***: `dfs[pat.charAt(j)][X]`!

We can use DFA to do this still because `X` is smaller than `j` and the DFA is partially completed!

You will find some interesting facts about `X`. For example, if `X` is incremented after one update, it could only be incremented by `1`; `X` might also be lower after one update (e.g., if the newly added char is `c` rather than `B`, `X` will become `0`).

Okay, take a deep breath! We have gone through everything about KMP now, and I believe (100%) that you are confident in writing the code by yourself.

```Java
public class KMP {

    private String pat;
    private int[][] dfa;

    public KMP(String pat) {
        this.pat = pat;
        int M = pat.length();
        int R = 256;
        dfa = new int[R][M];
        dfa[pat.charAt(0)][0] = 1;
        for (int j = 1, X = 0; j < M; j++) {
            for (int c = 0; c < R; c ++) {
                dfa[c][j] = dfa[c][X];
            }
            dfa[pat.charAt(j)][j] = j + 1;

            // Update X
            X = dfa[pat.charAt(j)][X]
        }
    }

    public int search(String txt) {
        int i, j;
        int N = txt.length(), M = this.pat.length();
        for(i = 0, j = 0; i < N && j < M; i++) {
            j = dfa[txt.charAt(i)][j];
        }
        if (j == M) return i - M;
        else return N;
    }

}
```

#### 1D Version with PMT

If you are a Chinese reader, you can also refer to [this post](https://www.zhihu.com/question/21923021/answer/281346746) for a new version which only uses a 1-D array. I will translate it into English later.

One hint I would give you is: though these two versions seem to differ a lot, they still share something (which plays a big role in KMP algorithm) in common. ***You will find that the the "PMT" mentioned in this version is exactly the restart state `X` is the 2D version!***
