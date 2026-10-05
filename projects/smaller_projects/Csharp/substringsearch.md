---
layout: default
title: Portfolio |  Substring Search
---

[Home](../../../../index.md) / [Smaller Projects](../index.md) /

# Substring Search
This code performs substring search using l-grams for matching limited character strings, such as DNA matching.

```csharp
using System;

public class SubstringSearch {

    // Returns the last index j where p[j] != t[i+j], or -1
    static private int LastUnequalChar(string p, Func<int, char> t, int i){
        for (int j = p.Length - 1; j >= 0; j--) {
            if (t(i+j) != p[j]) { return j; }
        }
        return -1;
    }

    // Returns the first index i where p == t[i..(i+p.Length)]
    static public int SubstringSearch(string p, Func<int, char> t, int n, int l){
        
        Dictionary<string, int> last = GetLGrams(p, l);

        int i = 0;
        int m = p.Length;

        int heuristic(string lgram){
            if (last.TryGetValue(lgram, out int pos))
                 return 1;
            return m + 1;
        }

        while (i <= n - m) {
            int j = LastUnequalChar(p, t, i);
            if (j == -1) { return i; }    // equal!
            if (i == n - m){ break;}

            // look ahead one l gram, using a char buffer
            // abaabcg
            //   abg
            //    ^^^
            char[] lookaheadBuffer = new char[l];
            for(int x = 0; x < l; x++){
                lookaheadBuffer[x] = t(i+m-l+1+x);
            }

            i += heuristic(new string(lookaheadBuffer));

        }
        return -1;
    }

    private static Dictionary<string, int> GetLGrams(string pattern, int l){

        Dictionary<string, int> dict = new();

        char[] buffer = new char[l]; //buffer used to construct the lgram strings efficiently

        int m = pattern.Length;

        // construct l grams,
        for(int i = 0; i <= m - l; i ++){
            for(int j = 0; j < l; j++){
                buffer[j] = pattern[i+j];
            }
            string lGram = new string(buffer);
            dict[lGram] = i;
        }


        return dict;
    }

    // Returns the optimal value of l for the given pattern length m
    static public int OptimalL(int m){
        if(m >= 750){
            return 6;
        }
        else if(m >= 192){
            return 5;
        }
        else if(m > 50){
            return 4;
        }
        else if(m >= 14){
            return 3;
        }
        else if(m >= 4){
            return 2;
        }
    
        return 1;
    }

}
```