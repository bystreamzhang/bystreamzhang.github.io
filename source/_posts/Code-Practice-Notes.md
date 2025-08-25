---
title: Code-Practice-Notes
date: 2025-08-25 10:31:49
tags:
    - Code
categories: Study
---

# 说明

刷题记录，为找工作做准备。

## LRC 001

```c++
class Solution {
public:
    int get_x_plus_b(int x, int *c){
        int res = 0, i = 0;
        while(x && i < 31){
            if(x | 1){
                if(res > INF - c[i]){
                    return -1;
                }
                res += c[i];
            }
            i++;
            x >> = 1;
        }
        return res;
    }
    int divide(int a, int b) {
        const int INF = (1 << 31) - 1; //虽然(1 << 31)是-21483648，但+1会变回2147483647，考虑底层二进制原理
        if(!b) return INF;
        int c[32];

        //int f = ((a < 0 && b > 0) || (a > 0 && b < 0)) ? -1 : 1;
        //a = (a < 0) ? -a : a;
        //b = (b < 0) ? -b : b;
        // 求c[i] = b^(i)
        /*
        b^i = b^(i - 1) * b = b^(i-1) * (2^0 + 2^1 + ... )
        考虑求出c[i]时同时维护d[i][j] = c[i] * 2^(j) = c[i] * 2^(j-1) + c[i] * 2^(j-1) = d[i][j-1] + d[i][j-1]
        */
        c[0] = 1;
        d[0][0] = 1;
        for(int j = 1; j <= 31; j++){
            d[0][j] = d[0][j-1] + d[0][j-1];
        }
        c[1] = b;
        for(int j = 1; j <= 31; j++){
            d[1][j] = d[1][j-1] + d[1][j-1];
        }
        for(int i = 2; i <= 30; i ++){
            c[i] = 0;
            for(int tb = b; b; b >>= 1){
                c[i] += 
            }
            c[i] = c[i-1] + c[i-1];
            if(i > (INF >> 1)) break;
        }
        int l = 0, r = INF;
        int mi;
        int prod = 0;
        while(l <= r){
            mi = ((r - l) >> 1) + l;
            prod = get_x_plus_b(mi, c);
            if(prod > a){
                r = prod - 1;
            }else if(prod == a){
                break;
            }else{
                l = prod + 1;
            }
        }
        return prod;
    }
};
```