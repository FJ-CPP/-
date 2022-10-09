[TOC]

# 字符串匹配DP

题解见注释。

## [LeetCode 10. 正则表达式匹配](https://leetcode.cn/problems/regular-expression-matching/)

> 给你一个字符串 s 和一个字符规律 p，请你来实现一个支持 '.' 和 '*' 的正则表达式匹配。
>
> '.' 匹配任意单个字符
> '*' 匹配零个或多个前面的那一个元素
> 所谓匹配，是要涵盖 整个 字符串 s的，而不是部分字符串。

```C++
bool isMatch(string s, string p) 
{
    int sl = s.size();
    int pl = p.size();
    // dp[i][j]表示s的前i个字符匹配p的前j个字符
    vector<vector<bool>> dp(sl + 1, vector<bool>(pl + 1));
    for (int i = 0; i <= sl; ++i)
    {
        for (int j = 0; j <= pl; ++j)
        {
            if (j == 0)
            {
                dp[i][j] = i == 0; // p的前0个字符只能匹配s的前0个字符
                continue;
            }
            if (i >= 1 && (s[i - 1] == p[j - 1] || p[j - 1] == '.'))
                dp[i][j] = dp[i - 1][j - 1]; // s的第i个字符与p的第j个匹配，因此整体能否匹配取决于s的前i-1个和p的前j-1个
            else if (p[j - 1] == '*')
            {
                // 使用这个*，那么必须保证p的第j-1个字符能够匹配s的第i个字符
                // 如果能匹配，则整体能否匹配取决于s的前i-1个字符能否与p的前j个字符匹配
                if (i >= 1 && (p[j - 2] == s[i - 1] || p[j - 2] == '.'))
                    dp[i][j] = dp[i - 1][j]; 
                // 也可以不使用这个*，即匹配0个*前面的那个字符
                // 则整体能否匹配取决于p的前j-2个字符能否匹配s的前i个字符
                dp[i][j] = dp[i][j] || dp[i][j - 2];
            }
        }
    }
    return dp[sl][pl];
}
```

## [LeetCode 44. 通配符匹配](https://leetcode.cn/problems/wildcard-matching/)

> 给定一个字符串 (s) 和一个字符模式 (p) ，实现一个支持 '?' 和 '*' 的通配符匹配。
>
> '?' 可以匹配任何单个字符。
> '*' 可以匹配任意字符串（包括空字符串）。
> 两个字符串完全匹配才算匹配成功。
>
> 说明:
>
> s 可能为空，且只包含从 a-z 的小写字母。
> p 可能为空，且只包含从 a-z 的小写字母，以及字符 ? 和 *。

```C++
bool isMatch(string s, string p) 
{
    int sl = s.size();
    int pl = p.size();
    vector<vector<bool>> dp(sl + 1, vector<bool>(pl + 1));
    for (int i = 1; i <= sl; ++i)
    {
        if (p[i - 1] != '*')
            break;
        dp[0][i] = true;
    }
    // dp[i][j]表示p的前j个能否匹配s的前i个字符
    for (int i = 0; i <= sl; ++i)
    {
        for (int j = 0; j <= pl; ++j)
        {
            if (j == 0)
            {
                dp[i][j] = i == 0; // p的前0个字符只能匹配s的前0个字符
                continue;
            }
            if (i >= 1 && (p[j - 1] == s[i - 1] || p[j - 1] == '?'))
            {
                dp[i][j] = dp[i - 1][j - 1];
            }
            else if (p[j - 1] == '*')
            {
                // 让它匹配空子串，则dp[i][j]为真取决于p的前j-1个字符能否匹配s的前i个
                dp[i][j] = dp[i][j - 1];
                // 让它匹配当前字符，则dp[i][j]为真取决于p的前j个字符能匹配s的前i-1个
                if (i >= 1)
                    dp[i][j] = dp[i][j] || dp[i - 1][j];
            }
        }
    }

    return dp[sl][pl];
}
```

## [LeetCode 72. 编辑距离](https://leetcode.cn/problems/edit-distance/)

> 给你两个单词 word1 和 word2， 请返回将 word1 转换成 word2 所使用的最少操作数  。
>
> 你可以对一个单词进行如下三种操作：
>
> 插入一个字符
> 删除一个字符
> 替换一个字符

```C++
class Solution {
public:
    int minDistance(string word1, string word2) 
    {
        typedef unsigned long long ull;
        int n1 = word1.size();
        int n2 = word2.size();
        vector<vector<ull>> dp(n1 + 1, vector<ull>(n2 + 1));

        // dp[i][j]表示word1的前i个字符转换成word2的前j个字符需要的最少操作数
        for (int i = 0; i <= n1; ++i)
        {
            for (int j = 0; j <= n2; ++j)
            {
                if (i == 0)
                    dp[i][j] = j; // 用word1的前0个字符转换成word2的前j个，只能插入j个字符
                else if (j == 0)
                    dp[i][j] = i; // 用word1的前i个字符转换成word2的前0个，只能删除i个字符
                else
                {
                    // word1的第i个字符与word2的第j个字符相等，则本轮无需操作，操作数依赖于word1的前i-1个匹配word2的前j-1个
                    // word1的第i个字符与word2的第j个字符不相等，则本轮可以选择三种操作:
                    // 1、如果删除第i个字符，则dp[i][j]=dp[i-1][j]+1，即只能用word1的前i-1个字符转换成word2，同时操作数+1（因为进行了删除）
                    // 2、如果替换第i个字符，则dp[i][j]=dp[i-1][j-1]+1，即转换成了word1的第i个字符与word2的第j个字符相等的情况，只是操作数要+1（因为进行了替换）
                    // 3、如果插入第i个字符，则dp[i][j]=dp[i][j-1]+1，即使用word1的前i个字符匹配word2的前j-1个，同时插入一个字符匹配word2的第j个字符
                    if (word1[i - 1] == word2[j - 1])
                        dp[i][j] = dp[i - 1][j - 1];
                    else
                        dp[i][j] = min(dp[i - 1][j], min(dp[i - 1][j - 1], dp[i][j - 1])) + 1;
                }
            }
        }

        return dp[n1][n2];
    }
};
```

## [LeetCode 115. 不同的子序列](https://leetcode.cn/problems/distinct-subsequences/)

> 给定一个字符串 s 和一个字符串 t ，计算在 s 的子序列中 t 出现的个数。
>
> 字符串的一个 子序列 是指，通过删除一些（也可以不删除）字符且不干扰剩余字符相对位置所组成的新字符串。（例如，"ACE" 是 "ABCDE" 的一个子序列，而 "AEC" 不是）
>
> 题目数据保证答案符合 32 位带符号整数范围。
>

```C++
class Solution {
public:
    int numDistinct(string s, string t) 
    {
        int sl = s.size();
        int tl = t.size();
        // dp[i][j]表示用s的前i个字符匹配t的前j个字符有多少个不同的子序列
        vector<vector<unsigned long long>> dp(sl + 1, vector<unsigned long long>(tl + 1));
        // 初始化dp数组
        for (int i = 0; i <= sl; ++i)
        {
            dp[i][0] = 1; // s的前i个字符匹配t的前0个，各有1种方案
        }
        for (int i = 1; i <= sl; ++i)
        {
            for (int j = 1; j <= tl; ++j)
            {
                // s的第i个字符与t的第j个字符不相等，则只能用s的前i-1个字符匹配t的前j个
                // s的第i个字符与t的第j个字符相等，则可以选择是否使用该字符
                // 如果使用，则当前方案数基于s的前i-1个字符匹配t的前j-1个
                // 如果不使用，则方案数基于s的前i-1个字符匹配t的前j个
                if (s[i - 1] != t[j - 1])
                    dp[i][j] = dp[i - 1][j];
                else 
                    dp[i][j] = dp[i - 1][j] + dp[i - 1][j - 1];

            }
        }
        return dp[sl][tl];
    }
};
```

